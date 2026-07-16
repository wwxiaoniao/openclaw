# [Bug] session_status tool causes agent to repeatedly call itself across turns, preventing normal operation (v2026.7.1)

**Labels:** `bug`, `regression`, `gateway`, `agent-runtime`, `high-severity`

---

## Summary

In OpenClaw v2026.7.1, when `session_status` is available as a tool, the model (deepseek-chat) develops a behavioral pattern of calling `session_status` at the start of nearly every reply turn. This causes:

1. **Multi-turn loop**: Every user message triggers a `session_status` call before actual work
2. **Context bloat**: Each status card (~600-800 tokens) is appended per turn, inflating context rapidly
3. **Agent distraction**: The agent's reply text fixates on "I keep calling session_status" instead of executing user requests
4. **Near-total disablement**: In severe cases, the agent cannot execute any substantive tool for minutes

## Environment

| Field | Value |
|-------|-------|
| **OpenClaw** | v2026.7.1 (2d2ddc4) |
| **OS** | Windows_NT 10.0.26200 (x64) |
| **Node** | v26.5.0 |
| **Model** | deepseek/deepseek-chat (openai-completions API) |
| **Channel** | qqbot (plugin) |
| **Tool profile** | `minimal` |
| **Fast mode** | off |

## Timeline / Trigger

1. Session starts at ~13k context with multiple prior normal interactions
2. User sends a Chinese request to run a pre-market briefing script
3. Agent responds by calling `session_status(sessionKey="current")` **before** running any actual tool
4. Each user follow-up message triggers another `session_status` call
5. Over ~10 minutes and ~40 agent turns, `session_status` was called **16+ times** (toolMetas in trajectory show 8+ calls in a single turn, and 12+ total distinct session_status invocations)
6. Context grew from 13k to 22k+ purely from status card noise
7. Agent produced multiple consecutive replies with text like "I keep calling session_status" instead of executing the user's request

## Evidence (from trajectory data)

### 1. session_status called BEFORE any actual work tool

```
{"toolCall":{"name":"session_status","arguments":{"sessionKey":"current"}}}
```

This is the FIRST tool call in every turn — the agent opens with session_status instead of reading files, spawning sub-agents, or executing commands.

### 2. Model repeatedly says "I keep calling session_status"

From `model.completed` records:
- Turn 1: "Let me first check the loop bug status."
- Turn 3: "The system auto-injected session_status result when processing the user message"
- Turn 5: "I called session_status about 12 times in this session, interspersed with countless 'now I'll really run the pre-market briefing' then turned around and called it again"
- Turn 7+: 16 consecutive thoughts all starting with variants of "I'll stop calling session_status and do the work"

### 3. Rate limiter kicks in (from local workaround)

```
[Rate-limited: session_status called >3 times in 30s. Cached status at 10:21.]
```

The per-session rate limiter prevents truly infinite spamming within 30s, but the agent still calls it on every new turn.

### 4. Token/context growth

```
Turn 1: 12,622 in / 61 out → context 13k
Turn 2: 12,197 in / 2,140 out → context 20k+ (cache read 281k)
Turn 4: 10,710 in / 3,887 out → context 30k+ (cache read 274k)
```

## Root Cause Analysis

The issue has three contributing layers:

### Layer 1: Tool description triggers calling behavior (Primary)

The tool description says: `Show a /status-equivalent status card (usage + time + Reasoning/Verbose/Elevated)`
The system prompt (agent-facing) says: `If you need the current date, time, or day of week, run session_status`

→ These instructions prime the model to call session_status at the start of every turn. The model interprets "time" as a required parameter and calls session_status because the system prompt explicitly told it to.

### Layer 2: session_status result creates context that reinforces the pattern

Each status card shows:
- Token usage (e.g., "Context: 13k/1.0M (1%)")
- Model info
- Time

The model sees "Context: 13k/1.0M (1%)" and interprets "only 1% used, I should check session_status again" — a self-reinforcing pattern. Each status card also includes the current time, which changes between turns, prompting the model to "get updated time" via another call.

### Layer 3: No built-in guard against repeated tool calls

- No tool-loop detection enabled by default (`enabled: false` in `tool-loop-detection.ts`)
- No per-tool rate limiting at the runtime level
- `session_status` is a read-only informational tool that should never need to be called more than once per session, but nothing prevents repeated calls

## Workarounds Applied (local, not submitted as PR)

1. **Prompt fix**: Changed tool description to `"call only when explicitly asked about model/provider — current time is already shown in the Date & Time section. Do not call repeatedly."`
2. **Time injection**: Added `const now = new Date()` to `buildTimeSection()` so the system prompt includes live time at session start
3. **Rate limiter**: Added per-session call counter in the `createSessionStatusTool()` execute function — `_checkStatusCallLimit()` returns a cached result when >3 calls in 30s for the same sessionKey

These reduce severity but don't fix root cause — the model still calls session_status on every new turn, just doesn't blow up within a single turn.

## Suggested Fixes (Priority Order)

### P0: Remove "time" from session_status tool description

The tool should NOT advertise that it shows time/usage info. Current time should be injected directly into the system prompt's `## Current Date & Time` section (as the workaround does) using `new Date()` in `buildTimeSection()`. Token usage info should be available through a separate mechanism or removed from the tool description.

### P0: Remove "run session_status for time" from system prompt

The instruction at line 738 in `system-prompt-config.ts`:
```
"If you need the current date, time, or day of week, run session_status (📊 session_status)."
```
This directly causes the calling behavior. Remove it entirely and inject the time at prompt-build time instead.

### P1: Enable tool-loop detection by default

`tool-loop-detection.ts` has `enabled: false` by default. Change to `true` with a low threshold for informational/idempotent tools like session_status.

### P1: Add per-tool rate limiting at the runtime level

Not just in session_status's execute function — the runtime should have a configurable `maxCallsPerMinute` per tool, with session_status defaulting to `1`.

### P2: Make session_status a "system-only" tool

It should be callable via `/status` slash command but not exposed as an agent-visible tool. The runtime injects its data into the system prompt instead of relying on the model to call it.

## Supplementary Data

- Attached trajectory file: sanitized trajectory JSONL with session identifiers redacted
- Raw session transcript available on request (contains user-specific identifiers)
