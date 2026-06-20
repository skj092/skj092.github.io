---
layout: post
title: "How a Coding Agent Really Works: Reverse Engineering Claude Code"
date: 2026-06-20
tags: [ai, agents, python, engineering, architecture]
---

*These are my notes from a deep study session on the OpenClaude codebase. I used Claude as a mentor to guide the learning — the questions, the struggle to understand, and the mental models are mine; Claude helped me articulate them clearly.*

---

I spent a session doing something I should have done months ago: sitting down with the OpenClaude codebase — the open-source reimplementation of Claude Code — and refusing to move on until I could explain every architectural decision from first principles.

Not just "what does this file do" but "why does this exist, what breaks if you remove it, and how would you rebuild it from scratch."

This post is the mental model I came out with. If you want to build a coding agent and actually understand what you're building, this is the reading I wish I had first.

---

## Why a Chatbot Is Not an Agent

The distinction sounds obvious but most people get it wrong in a subtle way.

A chatbot calls the LLM once and returns text. The limitation isn't just that it "can't take actions." The deeper limitation is that it has no way to work on tasks that require more than one step of reasoning — even if every step succeeds.

Consider: "Find the authentication bug in this codebase."

Even if you paste the entire codebase into the context, you've got two problems:
1. A 50,000-line codebase is millions of tokens. It doesn't fit.
2. Even if it fit, the LLM can't verify its own fix by running the tests.

An agent solves both problems differently than you'd expect. It doesn't load the whole codebase — it **searches for what it needs on demand**. Tools like `grep` and `read_file` are not just "capabilities." They're a form of selective memory retrieval. The agent never holds the whole codebase in context. It fetches only what's relevant at each step.

This is the first insight worth internalizing: **tools are not about power, they're about focus**.

---

## The Loop Is the Architecture

Every coding agent in existence — Claude Code, Cursor Agent, Aider, OpenHands — reduces to the same pattern:

```
while True:
    call LLM
    if LLM requested tools → execute them, feed results back
    if LLM gave text only → done
```

That's it. The entire architecture is in service of making this loop reliable.

The technical term is **ReAct**: Reason → Act → Observe. The LLM reasons about what to do, the agent acts (executes tools), the LLM observes the results. Repeat.

What's non-obvious is **who decides when the loop stops**. Not the agent code. The agent has no semantic understanding of "done." The loop exits when the LLM returns a response with no tool calls. The model decides. The agent just enforces the rules around that decision.

This means all the interesting engineering is: what do you do when the LLM's decision is wrong?

---

## The messages[] Array Is Everything

The agent's "memory" is a single array of messages. Every user input, every LLM response, every tool call, every tool result — all appended chronologically to this array.

The Anthropic API has one hard constraint: messages must strictly alternate between `user` and `assistant` roles. This creates an interesting problem. Tool results are not conceptually "user messages," but the API requires them to be wrapped as one.

So the actual structure for one tool-use iteration looks like this:

```
messages[0]  user       "Find the auth bug"
messages[1]  assistant  "I'll search for it" + tool_use(grep, "authenticate")
messages[2]  user       tool_result("auth.ts:47: if user = null")    ← manufactured
messages[3]  assistant  "Found it. Let me read the file" + tool_use(read_file)
messages[4]  user       tool_result("<file contents>")               ← manufactured
messages[5]  assistant  "Here's the fix."                            ← no tool call → exit
```

Tool results masquerade as user messages. This is not a leaky abstraction — it's the API contract. Understanding this makes the entire codebase make sense.

---

## The Real Cost of Context Windows

Here's the production failure mode nobody talks about in tutorials: your agent will hit the token limit.

A real coding session with 30+ tool calls generates enormous context. File reads return hundreds of lines. Bash output can be thousands of lines. After 20 iterations, your `messages[]` array might be 180,000 tokens on a 200,000-token model.

The naive solution — truncate old messages — loses the context that makes the agent useful. The LLM forgets what the user originally asked, what files it already read, what decisions it already made.

The production solution is **auto-compaction**: before each LLM call, estimate the token count. If you're approaching the limit, fork a separate (cheap) LLM call to summarize the old messages. Replace them with that summary. Continue with 10,000 tokens instead of 190,000.

```python
def maybe_compact(messages):
    if estimate_tokens(messages) < THRESHOLD:
        return messages

    # Keep original request + last 4 messages
    to_summarize = messages[1:-4]
    recent = messages[-4:]

    summary = cheap_model.summarize(to_summarize,
        "Preserve: key decisions, files found, current task state")

    return [messages[0], summary_message(summary), *recent]
```

You lose detail. The agent may re-read files it already read. That's acceptable. Continuity matters more than completeness.

The tradeoff nobody mentions: after compaction, the agent has no record of the exact edits it made. If something breaks later, it can't reconstruct what changed. This is why agents that edit code should also maintain a git-based snapshot — not for the LLM, but for the human to recover from bad agent edits.

---

## Three Ways the Loop Can Break

The happy path is the LLM calls tools, gets results, calls more tools, and eventually stops. Real sessions are not happy paths.

**The stuck loop.** The LLM tries `bash("npm test")`, gets `EACCES permission denied`, and tries again. And again. It "knows" this should work, so it keeps trying. Without a guard, you burn tokens forever.

The fix: track failure counts per `(tool_name, error_category)` tuple. If the same tool fails with the same error 3 times, stop the agent and surface the problem. A succeeded tool resets the counter.

```python
failure_counts = {}  # (tool_name, error_category) → count

if is_error(result):
    key = (tool_name, categorize_error(result))
    failure_counts[key] = failure_counts.get(key, 0) + 1
    if failure_counts[key] >= 3:
        raise ToolFailureLoopError()
```

**The wandering agent.** Every tool succeeds. The agent reads files, runs commands, makes changes. But it never finishes. `max_turns` is the only thing that catches this — not the failure guard, which requires failures, not just endless progress.

**The mid-thought stop.** The LLM returns `end_turn` but with text that looks like "Let me think about the best approach here..." — incomplete, not a final answer. The agent should detect this with heuristics and inject a synthetic "please continue" message, capped at 3 attempts.

These three failure modes are why the loop in `query.ts` is 2,240 lines. The happy path is 30 lines. The other 2,200 are handling every way things go wrong.

---

## Why Tools Run in Parallel (and When They Don't)

When the LLM calls five tools at once — read file A, read file B, grep for pattern C, glob for D, read file E — there's no reason to run them sequentially. All five are read-only. They can run simultaneously.

But when the LLM calls `edit_file("auth.ts")` and `bash("npm test")` — these must run serially. The edit must complete before the tests run. And if two tools edit the same file in parallel, the second write silently overwrites the first.

The classification isn't on the tool — it's on the **input**. `bash("cat file.txt")` is safe to parallelize. `bash("rm -rf ./build")` is not. Same tool, different answer. The tool itself makes this determination because only it has the domain knowledge.

```python
READ_ONLY_TOOLS = {"read_file", "grep", "glob", "list_dir"}

reads  = [t for t in tool_calls if t.name in READ_ONLY_TOOLS]
writes = [t for t in tool_calls if t.name not in READ_ONLY_TOOLS]

# Reads: parallel
with ThreadPoolExecutor() as ex:
    read_results = list(ex.map(execute, reads))

# Writes: serial
for tool in writes:
    execute(tool)
```

---

## The Permission System Is Not Optional

Every tutorial skips this. Every production agent needs it.

An agent that can run arbitrary bash commands with no permission model will eventually do something destructive. Not because the LLM is malicious — because the LLM makes mistakes. It misreads a path, misunderstands the task scope, over-reaches.

The right design: every tool call passes through a `can_use_tool()` gate before execution. The gate is separate from the tool — tools know how to execute, not whether they're allowed to.

```python
def can_use_tool(tool_name, input, mode):
    if mode == "read_only":
        return tool_name in READ_ONLY_TOOLS
    if mode == "interactive":
        return ask_user(f"Allow {tool_name}({input})?")
    if mode == "full_auto":
        return True
```

At minimum, implement `interactive` mode for anything that modifies files or runs shell commands. The user should see and approve writes before they happen.

---

## The Architecture in One Picture

```
User Input
    │
    ▼
CLI / UI (renders streaming tokens in real-time)
    │
    ▼
Session Manager (loads memory, builds system prompt, tracks costs)
    │
    ▼
Agent Loop ──────────────────────────────────────────────
    │                                                    │
    ▼                                                    │
Call LLM (streaming)                                     │
    │                                                    │
    ├── text tokens ──────────────► display in real-time │
    │                                                    │
    └── tool_use blocks                                  │
            │                                            │
            ▼                                            │
    Tool Executor                                        │
        ├── read tools ──► parallel                      │
        └── write tools ─► serial                        │
            │                                            │
            ▼                                            │
    Permission Gate                                      │
            │                                            │
            ▼                                            │
    Actual Tools (grep, read, bash, edit)                │
            │                                            │
            ▼                                            │
    Append results to messages[]                         │
            │                                            │
            └────────────────────────────────────────────┘
                         (loop again)
```

Every other feature — compaction, failure guards, rate limit retry, continuation nudge, MCP integration — is a layer on top of this core loop.

---

## What I'd Build First

If you're starting from scratch, this is the order that makes sense:

1. **The loop** — while(True), call LLM, check for tool_use, execute, append, repeat. 50 lines.
2. **Streaming** — yield tokens as they arrive. Real-time output matters for feel.
3. **Tool failure guard** — 3 strikes per (tool, error) pair. Prevents infinite loops.
4. **Context compaction** — without this, anything beyond 15 tool calls crashes.
5. **Permission gate** — before shipping to anyone else.
6. **Rate limit retry** — exponential backoff, don't count retries as turns.
7. **Parallel reads** — ThreadPoolExecutor for read-only tools. Latency win.

Items 1-4 give you a production-quality single agent. Everything after that is refinement.

---

## The Insight That Changed How I Think About Agents

After going through all of this, the clearest way I can state what a coding agent actually is:

**An agent is a loop that makes the LLM more reliable than the LLM is on its own.**

The LLM reasons. The loop handles every way the LLM's reasoning falls short — it stops infinite loops, manages context, retries infrastructure failures, detects incomplete stops, enforces permissions.

The better your loop, the more capable your agent — even with the same underlying model.

That's the thing nobody tells you when you start building. You don't improve agents by changing the model. You improve them by engineering the loop around the model.

---

## Production Checklist: Everything You Need Before Shipping

The sections above explain the concepts. This section is the implementation checklist — every feature you need before calling an agent production-ready. Use this when starting a new agent project.

### The Loop

- [ ] `while True` exits on `end_turn` with no tool calls
- [ ] `messages[]` strictly alternates `user` / `assistant` roles
- [ ] Tool results wrapped as `user` role messages
- [ ] Hard `max_turns` limit (default 50) — catches wandering agents

### Reliability

- [ ] **Tool failure loop guard** — track `(tool_name, error_category)` counts, exit after 3 consecutive identical failures, reset on success
- [ ] **Rate limit retry** — `try/except` on HTTP 429, exponential backoff (1s → 2s → 4s), do NOT increment `turn_count` on retry
- [ ] **Continuation nudge** — detect mid-thought `end_turn` with heuristics, inject synthetic "please continue", cap at 3 nudges
- [ ] **Abort signal** — `threading.Event()` checked in every tool and LLM call so Ctrl+C stops everything cleanly

### Context Management

- [ ] **Auto-compaction** — estimate tokens before each LLM call, compact at 80% of context limit using a cheap model (Haiku)
- [ ] **Large output truncation** — save tool output above 10k chars to disk, send LLM a file path pointer instead

### Tool System

- [ ] Strategy pattern — loop calls `execute_tool(name, input)` without knowing what tools do
- [ ] Read-only tools run in parallel via `ThreadPoolExecutor`
- [ ] Write tools run serially
- [ ] `execute_tool` catches all exceptions and returns error strings — the LLM needs to see what went wrong

### Safety

- [ ] `can_use_tool()` permission gate before every execution
- [ ] Dangerous bash pattern detection (`rm -rf /`, `curl | bash`, writes to `/etc/`)
- [ ] At minimum: `interactive` mode that asks user before writes; `read_only` mode for analysis tasks

### Memory & Context

- [ ] Load `MEMORY.md` from home dir and project dir into system prompt at startup
- [ ] Auto-inject `git status` and recent commits as project context

### Observability

- [ ] Log every tool call, result, token usage, and loop exit reason
- [ ] Save full transcript as JSONL after each session
- [ ] Track: turn count, compaction frequency, tool error rate, loop exit reason distribution

---

### Model Selection by Task

| Task | Model | Why |
|------|-------|-----|
| Main agent loop | `claude-opus-4-8` | Best reasoning for complex tasks |
| Context summarization | `claude-haiku-4-5-20251001` | Fast and cheap for mechanical work |
| Permission classification | `claude-haiku-4-5-20251001` | Simple yes/no decisions |
| Sub-agents | `claude-sonnet-4-6` | Good balance for parallel subtasks |

---

### Prompt to Give Claude When Building

Copy this when asking Claude to implement an agentic system. It encodes all the patterns above so you don't have to explain them each time:

```
Build a production-grade coding agent in Python using the Anthropic SDK.
Implement ALL of the following:

Agent Loop:
- while(True) loop that exits when LLM returns no tool calls (end_turn)
- messages[] maintained as single source of truth, strictly alternating user/assistant
- Tool results wrapped as "user" role messages (Anthropic API contract)
- Hard max_turns limit (default 50)

Reliability (all required):
- Tool failure loop guard: track (tool_name, error_category) failures,
  exit after 3 consecutive identical failures, reset counter on success
- Rate limit retry: try/except on RateLimitError, exponential backoff (1s/2s/4s),
  do NOT increment turn_count on retry — rate limit is infrastructure noise
- Continuation nudge: detect mid-thought stops using heuristics,
  inject synthetic "please continue" message, cap at 3 nudges
- AbortController: threading.Event() checked in every tool and LLM call

Context Management:
- Auto-compaction: estimate tokens before each LLM call, compact at 80% of limit
- Compaction: summarize messages[1:-4] using Haiku, replace with summary,
  keep first message + last 4 messages verbatim
- Large output: save tool output >10k chars to disk, send LLM a file path pointer

Tool System:
- Strategy pattern: loop calls execute_tool(name, input) without knowing implementations
- Read-only tools (read_file, grep, glob): run in parallel via ThreadPoolExecutor
- Write tools (bash, write_file, edit_file): run serially
- execute_tool catches all exceptions, returns error string so LLM sees what went wrong

Safety:
- can_use_tool() gate before every execution
- Dangerous pattern detection for bash (rm -rf /, curl|bash, writes to /etc/)
- Permission modes: read_only, interactive (ask user), full_auto

Memory:
- Load MEMORY.md from ~/.agent/ and project dir into system prompt at startup
- Auto-inject git status and recent commits as project context

Observability:
- Log every tool call, result, token usage, and loop exit reason
- Save full transcript as JSONL after each session

Tools to include: read_file, write_file, edit_file (string replacement),
bash, grep, glob, list_dir.
Use streaming for all LLM calls. Default model: claude-opus-4-8.
```
