---
layout: post
title: "Seams & Turns: Reverse-Engineering a Production Agent Harness"
date: 2026-08-15
tags: [ai, agents, engineering, architecture]
---

*A few months back I wrote about [how a coding agent really works](/blog/2026/06/20/how-a-coding-agent-really-works/) — the ReAct loop, the `messages[]` array, the failure guards. That post was about the loop. This one is about everything a loop needs around it before you'd trust it in production.*

*I spent a stretch going deep on DeepSeek Harness (`dsh`) — a real, currently-developed, plugin-based agent harness built on a framework called Cordis. It's the most complete answer I've found to a question the earlier post left open: what actually separates a while-loop around a chat completion from a system you'd let run unattended against a real codebase? I turned what I learned into a twelve-chapter course for myself, each concept paired with a small exercise so it would stick. This post is that course — the ideas, the diagrams, and every exercise, condensed from a much longer working document.*

---

## The task that runs through all of it

Everything below follows one concrete scenario. `checkout.spec.ts` fails about one run in six in CI. Nobody has touched that file in three months. You point an agent at the repository and type one sentence — *"find out why this test is flaky and fix it."* Twelve subsystems sit between that sentence and a merged fix that stays fixed. Each section below picks the story back up at the moment its subsystem becomes the thing standing in the way.

Here's the whole system at a glance, before the detail:

```
                 ┌──────────────────────────────────────────────┐
                 │        PLUGIN SUBSTRATE (Cordis)               │
                 │   every box below is a replaceable plugin      │
                 │                                                │
                 │   Agent loop  <── turn/step events ──>  Session│
                 │   (turn/step)                            log   │
                 │                                        (the    │
                 │   Tools <── tool/call, result ──>       only   │
                 │   +approval+sandbox                    truth) │
                 │                                                │
                 │   Model adapter <── assistant/chunk ──>        │
                 │   +streaming+compaction                        │
                 │                                                │
                 │   Subagents <── forked session ──>             │
                 │   +delegation+workflows                        │
                 └──────────────────────────────────────────────┘
                                      ▲
                                      │ chosen by
                          profiles → bundles → patches
```

Five boxes and one rule about how they talk to the log. Everything in this post is one of those boxes, or a rule governing it.

---

## Part I — Foundations

### 1. Turns, steps, and the session log

**9:14am.** You send the message. The driver claims it, opens a *turn*, and the model asks to run `pnpm test checkout.spec.ts`. Ten minutes and four tool calls later your laptop sleeps mid-step. When it wakes, the session doesn't restart from scratch and doesn't pretend nothing happened — it knows exactly which tool call never got an answer, because that's the only kind of fact its log can contain.

A **turn** opens when input is admitted and closes once nothing is owed. A **step** is one model request plus the tool calls its response caused — a turn is zero or more steps. The real event sequence:

```
turn/start
   │
   ▼
claim input ──► pre-step (reject | enter) ──reject──► turn/end (0 steps)
   ▲                  │ enter
   │                  ▼
   │            step/start
   │                  │
   │                  ▼
   │        request → stream → tool pipeline
   │                  │
   │                  ▼
   │              step/end
   │                  │
   └── tool owes another request, or new input arrived ── (no new turn/start)
                       │
                       │ nothing owed
                       ▼
                 turn-stopping (serial, no next())
                       │
                       ▼
                    turn/end
```

`agent/pre-step` is a **waterfall** — a listener can reject the claimed input outright, or rewrite it before it's entered. A reject, or an enter rewritten to nothing, still closes a durable turn that spent zero steps; the log records the attempt either way. The loop-back to "claim input" never appends a new `turn/start` — a turn is one open span holding as many steps as tools keep requesting.

Two invariants make this trustworthy under failure. **Model-visible ⟺ logged**: anything that reached a model request must be reconstructable purely by replaying the event log — no side-channel memory a resumed session can silently lose. And cancellation never leaves a dangling call: an undispatched tool call gets a synthetic result pairing it with `ABORTED_BEFORE_DISPATCH`, and a crash mid-tool-call is repaired the same way on reload, distinguishing "never started" from "outcome unknown" — the two demand different retry behavior, and conflating them risks re-running a non-idempotent command.

*Seen in dsh: `packages/core/agent-loop/src/agent.ts` (the entire driver in one file), `packages/core/session/src/surface.ts` (the projection behind `deriveMessages()`), `docs/agent-lifecycle.md`.*

**Exercises**

- [ ] **Warm-up** — Wrap a single model call in explicit `Turn`/`Step` objects with a status field instead of a bare function returning text. Done when you can print a full lifecycle as named state transitions, not just a final answer.
- [ ] **Core** — Add a `preStep` waterfall that can reject a claimed input or rewrite it. Done when a reject still produces a closed turn with zero steps.
- [ ] **Core** — Let a tool result trigger another model request inside the *same* turn, looping step → step without a new `turn/start`. Done when a turn with 3 tool-triggered steps logs exactly one `turn/start` and one `turn/end`.
- [ ] **Core** — Build an append-only log with `append()` and a `deriveMessages()` projector, plus a "replace" op that shadows old entries without deleting them. Done when replaying the raw log reproduces `deriveMessages()` exactly.
- [ ] **Advanced** — Add `AbortSignal` cancellation with a wake-latch: a wake arriving mid-teardown is queued and replayed exactly once. Done when firing cancel and an immediate follow-up concurrently always yields exactly one subsequent run.
- [ ] **Advanced** — Write a crash-repair pass for a truncated log (a tool call with no result). Done when it distinguishes "never started" from "outcome unknown" with two different synthetic results.

### 2. The plugin substrate

**Later that afternoon.** The fix works, but it took forty tool calls and eleven minutes. A teammate suggests a cheaper model for the exploration phase, saving the expensive one for the actual patch. Nobody touches the loop, the tool registry, or the log to make that possible — the model adapter is a plugin like everything else.

Cordis rests on a few ideas. A **plugin** is a function or class with `apply(ctx)`. A **context** is a repository of services, each claiming a stable key (`ctx.tools`, `ctx.llm`) — other plugins find a service by key, never by importing a concrete class. `inject` declares a dependency: a plugin naming a required service sits pending until that service exists, so load order is expressed as requirements, not manual sequencing. Events use one of four dispatch modes — `emit` (fire-and-forget), `parallel` (fan out, await all), `serial` (await in order), and **waterfall** (around-middleware: each listener gets the payload plus `next()`; call it to delegate, or return without it and you own the decision).

That last rule is enforced hard, for good reason:

```
EVERY LISTENER CALLS next()
caller ──► listener 1 ──► listener 2 ──► listener 3 ──► final result

LISTENER 2 FORGETS next()
caller ──► listener 1 ──► listener 2 ✕ short-circuits here
                                       listener 3 and the result are never reached
```

The idea that makes all of this composable is that **registrations are effects** — every tool, listener, or adapter goes through `ctx.effect()`, which returns a disposer. Unmounting a plugin reverses every effect it created, in reverse order, automatically. That's the concrete difference from a plain DI container or a bare `EventEmitter`: there, cleanup is a convention you remember; here, the registration *is* the disposer contract, which is what makes hot-reloading one plugin, or composing a different product from the same packages, safe rather than a source of leaks.

*Seen in dsh: `docs/cordis-primer.md`, `docs/cordis-tutorial/06-composition-and-hmr.md`.*

**Exercises**

- [ ] **Warm-up** — Implement `ctx.effect(fn)` storing a disposer, and `unmount(plugin)` calling disposers in reverse order. Done when unmounting a plugin leaves zero trace while a sibling's effects survive.
- [ ] **Core** — Write `ctx.waterfall(event, ...args)` with `(...args, next)` listeners. Done when you can demonstrate the "forgets `next()`" bug silently dropping a downstream listener, then fix it.
- [ ] **Core** — Build `inject`-driven load ordering: a plugin naming a dependency stays pending until it exists. Done when registering the dependency *after* the dependent still activates it correctly.
- [ ] **Advanced** — Make a plugin safely reloadable (unmount + remount). Done when reloading twice in a row leaves the registry identical to a single mount.
- [ ] **Advanced** — Implement all four dispatch modes over one registration primitive, with one test proving each mode's distinct contract.

### 3. Capability seams

**Next tool call.** The failing test spawns a real browser against a real local port. Locally that's plain `bash`; in CI, the same tool call needs to run inside whatever confined shell CI provides. The tool that says "run this command" never needs to know which one it's talking to.

A **capability seam** has three roles: a **Service Definition** (in `dsh` always an abstract class, never a bare interface, because it can carry shared behavior and brands its context key), one or more **Service Providers** (concrete subclasses), and one or more **Consumers** (usually a model-facing tool). The canonical example is `dsh-shell`: `ShellExecutor` is the abstract class; `dsh-bash-local` and `dsh-bash-sandbox` are providers; `dsh-tool-bash` is the consumer. Cordis enforces one implementation per context — a second provider throws on mount, because the whole point is that a caller never asks which provider is active.

One rule threads through every seam: **explicit defaulting, never a hidden fallback**. A provider exposes `resolve(request): Spec`, filling in defaults explicitly, rather than a `?? default` buried inside `run()`. The payoff shows up at the scale of an entire execution world: because `dsh-bash-local`, the terminal, and the language server never touch the filesystem or process APIs directly — only through `ctx.fs` and `ctx.subprocess` — pointing those two providers at a remote sandbox moves Bash, PTY, and LSP execution together, with zero forks of any consumer package.

*Seen in dsh: `packages/shell/shell/src/index.ts`, `docs/capability-seams.md`, `docs/glossary.md`.*

**Exercises**

- [ ] **Warm-up** — Define an abstract `Clock` Service Definition and one provider. Done when a consumer only ever imports the `Clock` type.
- [ ] **Core** — Enforce one implementation per context: a second provider registration throws, naming both providers.
- [ ] **Core** — Model the request/spec `resolve()` split for a toy HTTP fetcher. Done when two provider configs with different default timeouts produce two different resolved specs from an identical request.
- [ ] **Advanced** — Build two providers for a `KeyValueStore` seam (in-memory, file-backed) and one consumer that never references either concretely. Done when swapping providers is a one-line config change with zero consumer edits.

---

## Part II — The loop in motion

### 4. Tools and guarded execution

**Three tool calls in.** The model proposes `rm -rf node_modules && pnpm install` to rule out a dependency issue. That's a real command with real consequences, and it doesn't run the instant the model asks — it passes through a decision point first, the same one every tool call passes through.

```
tool call
   │
   ▼
pre-execute (waterfall) ── deny ──────────────► never runs
   │            └─ ask ──► approval ── rejected ──► never runs
   │ allow                    └─ allowed-once ─┐
   ▼                                            │
guards (monotonic, deny-only) ◄─────────────────┘
   │
   ▼
execute (waterfall: timeout / retry / metrics wrap the dispatch)
   │
   ▼
tool body
   │
   ▼
post-execute (waterfall) ── block ──► feedback becomes isError
   │ accept
   ▼
tool/result (durable)
```

The registry holds **global** tools and tools **scoped** to one agent, keyed by that agent's own identity — a scoped tool **shadows** a same-named global one. `restrict(filter)` narrows the *inherited* global set by intersection but never touches a scope's own registrations, so a delegated child always keeps the tools it answers through. Guards are deliberately **monotonic** — they can only deny, never restore permission a waterfall already denied, so registration order can never accidentally re-open a closed door. Approval itself is a real seam: its outcome is a closed set (`allowed-once`, `rejected`, `cancelled`, `unavailable`), and a missing or throwing approval fails closed to denial, never open.

A tool's schema joins the prompt through an **explicit allowlist** — name, description, parameters, nothing else — so a host-only field like `execute` or `timeoutMs` can never leak onto the wire by accident. And a tool's UI render intent (`generic`/`terminal`/`diff`/`search`) is decided when the tool is authored, because the presenter must be a pure function of its arguments and result — it has to produce identical output whether it's rendering live or replaying a session from months ago.

*Seen in dsh: `packages/core/tools/src/index.ts`, `packages/interaction/user-approval/src/index.ts`, `docs/cookbook/adding-a-tool.md`.*

**Exercises**

- [ ] **Warm-up** — Build a tool-definition type with model-facing and host-only fields, and a `schemas()` projector that's allowlist-safe by construction.
- [ ] **Core** — Implement a scoped registry: `register`/`get` (scoped-over-global) and `restrict()` that intersects without hiding a scope's own tools.
- [ ] **Core** — Implement the three-stage pipeline as waterfalls plus monotonic guards. Done when a deny can't be reversed by a later "allow" guard.
- [ ] **Core** — Build a minimal approval seam with the closed outcome set and a durable per-session `ask`/`never` policy. Done when `never` deterministically rejects without invoking an answerer.
- [ ] **Advanced** — Write pure `presentCall`/`presentResult` for a mock diff-producing tool. Done when two "replay sessions" produce byte-identical output from the same logged arguments.

### 5. The model adapter and streaming

**Watching the terminal.** Text appears token by token as the model reasons about a race condition in a `beforeEach` hook, then a visibly different block appears — a tool call being assembled, not prose. The cheap and expensive models from chapter 2 both produce exactly these same shapes, which is the only reason swapping between them was ever a config change.

Every provider normalizes down to seven chunk types and one terminal outcome: `block-start`, `text-delta`, `reasoning-delta`, `tool-call-delta`, `block-end`, `usage`, and exactly one `finish {kind: 'ok' | 'error' | 'aborted'}`. Every adapter resolves to that shape rather than throwing across the stream API — which is what lets one shared `BlockAssembler`, not each adapter, build the final message and feed both live history and replay. `llm/stream` is itself a waterfall, so a plugin can wrap the call entirely — for caching, logging, or routing — by returning a value without calling `next()`. Failure recovery is a separate waterfall, `agent/request-error`, fired with a normalized failure code (`AUTH`, `RATE_LIMIT`, `CONTEXT_WINDOW_EXCEEDED`, `TRANSPORT`...); everything downstream, including compaction's overflow repair, dispatches on those codes rather than parsing provider-specific error text.

*Seen in dsh: `packages/llm/llm/src/types.ts`, `packages/llm/llm-deepseek/src/adapter.ts`.*

**Exercises**

- [ ] **Warm-up** — Implement a fake `stream()` emitting the chunk sequence with artificial latency. Done when an aborted signal mid-stream yields `finish {kind: 'aborted'}`, never a thrown exception.
- [ ] **Core** — Build the chunk-to-message reducer as its own testable unit, independent of any network code.
- [ ] **Core** — Register an `llm/stream` listener returning a cached response for a fixed prompt without calling `next()`, alongside a pass-through listener for everything else.
- [ ] **Advanced** — Point your fake adapter's shape at a real SSE endpoint. Done when a dropped connection mid-stream still resolves to `finish {kind: 'error'}`.
- [ ] **Advanced** — Implement a request-error recovery waterfall: one listener retries on a simulated overflow code with trimmed history, another gives up on a simulated auth error.

### 6. Context engineering

**Forty minutes in.** The transcript is enormous — a dozen failed hypotheses, five full test runs, three thousand-line file reads. The next request would blow the context window. Something has to give, without losing the thread of what's already been ruled out, and without quietly rewriting history in a way nothing downstream can account for.

The system prompt is a **registry**, not a template — plugins contribute ordered sections, tool schemas, and {% raw %}`{{variable}}`{% endraw %} references, scope-aware just like tools. Compaction has two triggers (proactive pressure, reactive context-overflow) and a two-stage strategy: first a deterministic, model-free pass shrinks oversized tool results; only if that's insufficient does it summarize the *oldest whole span* via one real model call, replaying that span's exact original prefix so a provider's cache is reused right up to the final instruction.

```
raw log (nothing deleted):
[m1] [m2] [m3] [m4] [m5] [m6] [m7] [m8]
      └────────────┬────────────┘
      compaction/start … summary … end
      (log-only, NOT model-visible — a lock, not context)

what the model sees (deriveMessages()):
[m1] [       summary        ] [m6] [m7] [m8]
      ▲
      user/message · surfaceOp: replace  (ordinary, model-visible, logged)
```

The bracket events (`compaction/start`, `summary`, `end`) are bookkeeping for crash recovery — an unmatched `start` is a detectable orphaned lock. But they are deliberately *not* model-visible; the model only ever sees an ordinary logged message doing the replacement. That split is "model-visible ⟺ logged" applied to something more complex than a single fact.

*Seen in dsh: `packages/core/system-prompt/README.md`, `packages/compaction/compaction-basic/README.md`.*

**Exercises**

- [ ] **Warm-up** — Build a prompt section registry with ordered concatenation and strict {% raw %}`{{var}}`{% endraw %} interpolation that throws on an unknown reference.
- [ ] **Core** — Implement `compactIfNeeded(history, budget)` with logged start/summary/end brackets. Done when replaying only the event log reconstructs the exact final visible history, and a simulated crash between start/end leaves a detectable orphaned lock.
- [ ] **Core** — Implement a tool-result pruner measured by Unicode code point, not UTF-16 unit. Done when a surrogate-pair emoji at the truncation boundary is never corrupted.
- [ ] **Advanced** — Make your summarization call replay the exact original prefix of the span being compacted, verified against a mock "cache hit" flag.
- [ ] **Advanced** — Build an ephemeral "current time" context injector that only fires after a refresh interval, and survives a simulated compaction shadowing everything before it.

---

## Part III — Scaling out

### 7. Delegation and subagents

**Meanwhile.** While the main investigation keeps digging into the race condition, you also want to know: are there other flaky tests in the suite with the same pattern? Worth answering in parallel rather than making the main investigation context-switch onto it.

`ctx.subagents` is one Service Definition with many providers behind it — spawning a fresh child with empty history, forking a child seeded only with the parent's *completed* turns (never its own in-flight, unbalanced tool-calling turn), or driving an out-of-process agent over a protocol. Delegation depth is modeled as **data, not scope**: a persisted, monotone `delegationDepth` and a runtime override, both checked against a cap — scope answers "which registrations does this agent see," lineage answers "how did this agent come to exist," and conflating them would make depth policy leak into tool visibility.

```
Parent agent (depth 0, policy: ask)
        │
        ▼
   ctx.subagents (one definition, many providers)
     /                              \
spawn-in-process              fork-in-process
(fresh, empty history)   (seeded: completed turns only)
     \                              /
        ▼                        ▼
      Child agent (depth 1, policy forced: never)
                │
                ▼
      attempted depth 2 ── blocked: exceeds depthLimit
```

Every delegated child's approval policy is pinned to `never`, and the parent's sandbox override is snapshotted at spawn time — a denied action inside a child fails deterministically instead of hanging on an interactive prompt nobody is watching, because there's no human on the other end of a subagent's approval channel.

*Seen in dsh: `packages/subagent/subagent/README.md`, `subagent-spawn-in-process/`, `subagent-fork-in-process/`.*

**Exercises**

- [ ] **Warm-up** — Build a second in-process "mock-remote" provider alongside a spawn provider. Done when the same call site works unmodified against both, selected only by name.
- [ ] **Core** — Enforce `depthLimit`, with depth persisted so it survives a resume. Done when a child-of-child-of-child delegation is refused with a clear error.
- [ ] **Core** — Implement the fork variant: seed a child only with the parent's completed-turn prefix. Done when forking mid-tool-call never hands the child a dangling tool call.
- [ ] **Advanced** — Pin every delegated child's approval policy to `never` regardless of the parent's own policy. Done when a denied child action fails immediately, never hanging.
- [ ] **Advanced** — Make the registry reject an unsupported delegation request (advertised `depthLimit`/`toolFilter`) before the provider ever creates a child.

### 8. Sandboxing and the shared execution world

**Back on the delegated search.** It wants to run `grep` and `find` across the whole repository — fine — but it's a second, less-supervised line of execution than your main session. Whatever confines the main agent's shell has to confine this one too, without a second implementation to keep in sync.

`ctx.sandbox.confine(argv, policy)` returns wrapped argv, or throws — confinement **fails closed**, never falling back to running unconfined. Policy rides the individual call, not the provider instance, so two consumers can run concurrently under two different modes against the same backend. This is deliberately a **same-world** boundary (shares the host kernel), not a container/VM substitute. The E2B integration is the concrete proof of the seam architecture's whole claim: because Bash, the terminal, and the LSP client only ever go through `ctx.fs`/`ctx.subprocess`, mounting the E2B filesystem and subprocess providers moves all three into a remote sandbox together, with no code changes to any of them.

*Seen in dsh: `packages/sandbox/`, `packages/e2b/`, `native/README.md` (Landlock).*

**Exercises**

- [ ] **Warm-up** — Write `confine(argv, policy)` returning wrapped argv or throwing. Done when a test proves it throws rather than returning unwrapped argv when confinement can't be established.
- [ ] **Core** — Run two concurrent calls under two different sandbox modes against one shared provider instance without interference.
- [ ] **Advanced** — Define minimal `fs`/`subprocess` seams, a local-disk implementation, and an in-memory "remote" one. Done when swapping providers in config alone changes where files live, with zero consumer changes.
- [ ] **Advanced** — Model a self-restrict-then-exec wrapper (the Landlock pattern) that restricts its own privileges before "exec-ing" the real command, and cannot be widened afterward.

### 9. Guardrails

**Attempt six.** The model reruns `pnpm test checkout.spec.ts` with the exact same arguments as attempt five, having changed nothing in between. Nobody told it to stop — but something in the loop notices the pattern before you do, and says so, without ever taking the wheel away.

Neither guardrail in `dsh` is a new capability seam — both are plain plugins on extension points that already exist. A **repeat-call guard** tracks consecutive calls with the same tool name and canonicalized arguments per agent; at escalating thresholds it injects an advisory message, never blocking. Excluded bookkeeping tools don't reset the chain when interleaved, and a *denied* call still counts — a model looping on a call that keeps getting denied is still looping. A **timeout guard** is a single waterfall wrapper racing a tool-declared `timeoutMs` against a fused abort signal — purely cooperative, since a tool that never checks its signal doesn't actually stop just because the deadline passed.

*Seen in dsh: `packages/guard/`.*

**Exercises**

- [ ] **Warm-up** — Detect N identical consecutive calls (deep-sorted canonicalized arguments) per agent, at configurable thresholds.
- [ ] **Core** — Inject an escalating advisory through a post-execute-style hook instead of denying the call outright.
- [ ] **Core** — Thread an abort signal into a slow mock tool and race it against a per-tool deadline. Done when you can prove which side won by signal identity, not result shape.
- [ ] **Advanced** — Add an exclusion list of bookkeeping tools that don't reset the chain, and confirm a denied call still advances it.

### 10. Orchestration and the Ralph loop

**The fix lands.** Now there are three more flaky tests in the same suite, unrelated to each other, each needing its own from-scratch investigation. One long session holding all three in its head at once is worse than three short, focused ones — as long as each one can tell the next what it already ruled out.

```
Round 1 (fresh child) ──handoff #1──► Round 2 (fresh child) ──handoff #2──► Round 3 (fresh child)
        │                                     │                                     │
        └────────── shared workspace (filesystem), persists across every round ─────┘

           ✕ no conversation carried over between rounds — only workspace + one capped handoff
```

The workflow engine executes an orchestration script (`agent()`/`parallel()`/`pipeline()`) that a model can author at call time, isolated in a worker thread — not a security boundary, just host-loop isolation. The **Ralph loop** is a clean example of composition over special-casing: it's *just* a plugin calling the same engine with a fixed, non-model-authored script. Each round spawns one genuinely fresh child receiving only the immutable objective, the round cap, and the previous round's bounded structured handoff (`status`, `summary`, `evidence`, `nextSteps`, `blockerText`). Completion is self-declared by the worker — there's no independent verifier checking that claim against reality in the current system, a real limitation worth building past.

*Seen in dsh: `packages/workflow/`, `docs/glossary.md` (Ralph entries).*

**Exercises**

- [ ] **Warm-up** — Implement `parallel()` and `pipeline()` over a fake `agent()` call. Done when `pipeline()` genuinely overlaps stages across items while `parallel()` is a real barrier.
- [ ] **Core** — Enforce that a workflow's result must be plain, serializable JSON, with a named fatal error category on violation.
- [ ] **Core** — Build a Ralph-style bounded-handoff loop: fresh child per round, only a capped JSON report carrying forward.
- [ ] **Advanced** — Make one item's stage throw a "fatal" category that escapes a whole pipeline, while a "recoverable" category only nulls that item's slot.
- [ ] **Advanced** — Add an independent verifier round that checks a worker's self-declared "complete" against actual workspace state before accepting it.

---

## Part IV — Product

### 11. Profiles, bundles, and patches

**Rolling this out.** The same agent needs to run two different ways: interactively in your editor while you watch, and headlessly in CI with no one watching at all. Same tools, same model, same guardrails — just a different shell around them, and no fork in the code.

A **bundle** is a patch file plus the code it mounts, each row carrying a stable `id`. A **profile** is an ordered stack of bundles plus the user's own patch. Composition applies strictly: each bundle's patch in order, then the profile's own patch, then a home-level patch, then any one-off overlay. A patch is **id-targeted** — it replaces a row's entire config, no deep merge, or inserts new rows; an unmatched id just warns.

```
dsh-base bundle        fs-sandbox: cwd=/workspace
        │  applies over
        ▼
profile patch (web)     fs-sandbox: unchanged
        │  applies over
        ▼
home-level patch        fs-sandbox: cwd=/tmp/ci   ← replace id: fs-sandbox
        │  applies over
        ▼
--patch overlay         fs-sandbox: unchanged
        │
        ▼
composed tree           fs-sandbox: cwd=/tmp/ci   (home patch wins — applied last)
```

`dsh --profile web --dump-config` composes the same layers offline and prints the result, each row commented with the layer that contributed it. Because the model adapter, the tool registry, the session log, and the agent loop are each just an id-tagged row, none of them is structurally privileged — every one is replaceable by a later-layer patch, with zero code fork.

*Seen in dsh: `packages/bundle/base/cordis.patch.yml`, `packages/preset/README.md`.*

**Exercises**

- [ ] **Warm-up** — Represent your plugin list as `{id, config}` rows instead of a hand-written boot script.
- [ ] **Core** — Implement `composeEntries(baseRows, ...patchLayers)` where a later layer's matching id fully replaces the earlier one. Done when an unmatched-id patch only warns, never throws.
- [ ] **Core** — Render your composed tree back out with a provenance comment naming which layer contributed each row.
- [ ] **Advanced** — Let a "group" of rows declare `isolate: true` and get its own private service instance, invisible to sibling groups.
- [ ] **Advanced** — Add a host-conditional `disabled` predicate so one shared patch file mounts different rows per environment with zero branching in the consumers.

### 12. Observability, replay, and evaluation

**Three months later.** Someone asks why the agent decided the fix was `await`-ing a promise instead of increasing a timeout. The engineer who ran it has moved teams. The only honest answer is in the log.

```
the same code change
        │
   ┌────┴────┐
   ▼         ▼
unit tests   golden transcript (real run)
100% coverage    │
   │         replay vs. fixture
all green ✓      │
   │         byte-for-byte mismatch
ordering bug     │
ships anyway  caught before merge
```

If the session log is the only source of truth, the claim worth testing is "replaying the log reproduces the product's real behavior" — not merely that handlers return the right value in isolation. That's why a keyless snapshot of a real, runnable transcript outranks a high coverage number as evidence: a file can hit 100% coverage while still getting message ordering or replay fidelity wrong. Fixtures have to be deterministic (no live clock, no randomness, no real network) and built from a real runnable example, not a mock agreeing with itself.

**Exercises**

- [ ] **Warm-up** — Record one real end-to-end run's event log as a fixture; assert a fresh replay reproduces it exactly.
- [ ] **Core** — Write a file with 100% line coverage that still has a live ordering bug, then write the transcript test that actually catches it.
- [ ] **Core** — Remove every clock/random/network dependency from a harness slice. Done when two runs on two different simulated "days" produce byte-identical transcripts.
- [ ] **Advanced** — Feed a truncated log (mid-tool-call) into your repair pass and assert the resumed session can take one more real turn without error.

---

## The capstone: build one yourself

The flaky test got fixed by chapter 4, but notice everything that had to be true for that fix to be *trustworthy*: resumable after a crash, safe to run unattended, cheap enough to run for eleven minutes without melting a budget, explainable three months later to someone who wasn't there. None of that came from a better prompt. It came from the boxes in the diagram at the top of this post, built deliberately instead of accreted by accident.

If you want to actually build the muscle rather than just read about it, here's the order that makes sense, mirroring the checklist style from [the earlier post](/blog/2026/06/20/how-a-coding-agent-really-works/):

1. **Core loop, one real tool, real streaming** — chapters 1, 4, 5. A turn/step machine against a real model API with real streaming, one tool running through a guarded pipeline.
2. **Move it onto a plugin substrate** — chapters 2, 3. The registry, adapter, and log become plugins registering through an effect system, not hardwired calls.
3. **Add context management** — chapter 6. Prompt assembly plus token-budget compaction so a long task doesn't blow the window mid-task.
4. **Make it safe** — chapters 4, 8, 9. Sandboxed execution, an approval gate, at least one loop-hygiene guard.
5. **Add delegation** — chapters 7, 10. One subagent provider, used for something genuinely parallelizable.
6. **Express it as configuration** — chapter 11. Id-tagged rows composed through bundle/profile/patch layers, with a small introspection dump.
7. **Ship it on a real task** — chapter 12. Pick one real task — triage and fix failing tests in a small repo, research and summarize several sources, migrate a config format across a codebase — run the harness end to end, and write a transcript-based test proving it worked.

Items 1–4 give you something you'd trust with real stakes. Everything after that is what makes it scale past one task and one person watching.
