---
title: "How Claude Code Manages Infinite Conversations in a Finite Context Window"
description: "A source-level deep dive into Claude Code's three-tier context compaction system — microcompact, full compact, and session memory compact — and how it preserves critical context while staying within token limits."
pubDate: 2026-04-07
tags: ["Claude Code", "Anthropic", "AI Agents", "Context Window", "Architecture"]
---

Claude Code conversations have no turn limit. You can work for hours — reading files, running tests, debugging, iterating — and the conversation just keeps going. But the model has a fixed context window. At some point, the accumulated messages exceed what the model can process in a single API call.

The system needs to compress the conversation without losing critical context. Here's how it works, from the source code.

## The Problem

The naive approach is truncation: drop old messages when the window fills up. This fails immediately. A conversation about building an authentication system might reference a design decision from 50 turns ago. Truncate those turns and the model forgets the decision, re-asks the question, or contradicts what it said earlier.

A better approach: summarize. Replace the old messages with a summary that preserves the essential information. But summarization introduces its own problems:

- **What to preserve?** File paths, code snippets, user preferences, error resolutions, pending tasks — all matter. A generic "summarize this conversation" prompt loses critical details.
- **When to trigger?** Too early wastes context window. Too late risks hitting the hard limit and failing the API call.
- **What about the cache?** Anthropic's API caches the prompt prefix. Compaction replaces all messages, invalidating the cache. Every token in the new prompt is a cache miss — expensive.
- **What if the summary itself is too long?** If the conversation is so large that even the compaction request exceeds the context window, you need a fallback.

Claude Code solves these with a three-tier system. Microcompact clears stale tool results without calling the model. Full compact summarizes the entire conversation with a dedicated model call. Session memory compact uses pre-extracted notes to skip the summarization call entirely. Each tier is progressively more aggressive and more expensive.

## When to Compact

### The Threshold

Auto-compact fires when the conversation's token count exceeds a threshold. The threshold is calculated as:

```
effectiveWindow = contextWindow - max(maxOutputTokens, 20_000)
autoCompactThreshold = effectiveWindow - 13_000
```

For a 200K context window model, this works out to roughly 167K tokens. The 20K reserve ensures the model has room to generate the summary. The 13K buffer provides headroom — the system checks BEFORE each API call, so the actual token count may grow by a full model response between checks.

The threshold can be overridden via environment variables for testing. A percentage-based override lets you trigger compaction earlier (useful for observing the system's behavior on shorter conversations).

### Token Counting

The canonical function for context size is `tokenCountWithEstimation`. It works in two parts:

1. **Find the last API response**: Walk backward through messages to find the most recent assistant message that has a `usage` field (reported by the API). This gives the exact token count at that point in the conversation.

2. **Estimate new messages**: For any messages added AFTER the last API response, estimate their token count. Text blocks use a rough `length / 4` heuristic (one token per ~4 characters). Images and documents get a flat 2,000-token estimate. Tool use blocks count the tool name plus JSON-serialized input. The total estimate is padded by 4/3 (33% conservative buffer). Add this to the API-reported count.

A subtlety: when the model makes multiple parallel tool calls, each becomes a separate assistant message interleaved with tool results. The messages look like: `[..., assistant(id=A), toolResult, assistant(id=A), toolResult, ...]`. All of these share the same message ID because they came from one API response. The token counter must walk back to the FIRST message with matching ID to anchor correctly — stopping at the last one would miss the interleaved tool results and undercount.

The total context count includes input tokens, cache creation tokens, cache read tokens, and output tokens. This represents the actual context window consumption, which is what matters for threshold comparison. Using only `input_tokens` would undercount because cached tokens still occupy the window.

### The Circuit Breaker

If compaction fails three times consecutively, auto-compact stops trying. This circuit breaker prevents runaway API costs. Before the breaker, telemetry showed 1,279 sessions with 50+ consecutive failures, wasting approximately 250,000 API calls per day. The breaker resets on any successful compaction.

### Recursion Guards

Auto-compact skips triggering when the query source is `compact` (would deadlock — compaction triggering compaction) or `session_memory` (would deadlock — memory extraction happens in a forked subagent that shares the token counter).

## Tier 1: Microcompact

Microcompact is the cheapest intervention. No model call. No summarization. It just clears old tool results that the model no longer needs, reclaiming tokens.

### Time-Based Clearing

The API prompt cache has a TTL of roughly one hour. When the user returns after an idle period, the entire cached prefix is gone — every token will be re-processed anyway. This is the ideal time to clear stale tool results, because there's no cache to preserve.

The trigger:

```
gap = (now - lastAssistantMessage.timestamp) / 60_000
if gap > gapThresholdMinutes (default: 60):
  clear old tool results
```

"Clear" means replacing the content of `tool_result` blocks for compactable tools (file reads, shell output, grep results, glob results, web fetches, web searches, edits, writes) with the text `[Old tool result content cleared]`. The system keeps the N most recent results (default: 5) and clears the rest.

This is a mutation of the message array. The cleared results are gone. But since the cache was already expired, there's no cost — the full conversation will be re-tokenized regardless.

### Cached Microcompact

When the prompt cache is still warm, mutating messages would invalidate the cached prefix. Instead, the system uses the API's `cache_edits` feature to delete tool results server-side. The local message array stays unchanged, but the API receives a `cache_edits` block that instructs the server to remove specific tool results by their cache reference IDs.

The state machine tracks three things:

- **registeredTools**: Set of all tool_use IDs seen (deduplicated)
- **toolOrder**: List of tool_use IDs in encounter order (FIFO for deletion priority)
- **deletedRefs**: Set of IDs already deleted (prevents re-deletion)

The logic:

```
activeTools = toolOrder filtered by NOT in deletedRefs
if activeTools.length < triggerThreshold (default: 12):
  return (not enough tools to justify clearing)

toDelete = activeTools[0 .. activeTools.length - keepRecent]
for each id in toDelete:
  add to deletedRefs
  create cache_edit { type: "delete", cache_reference: id }

queue as pendingCacheEdits (applied at API layer)
```

The cache edits are "pinned" — once queued, they're re-sent on every subsequent API call for as long as the cache hit persists. This is necessary because cache edits are relative to the cached prefix, not absolute. If the server cache is hit, the pinned edits tell it which blocks to skip.

If the cache expires (detected by a drop in `cache_read_input_tokens`), the pinned edits become stale. The system falls through to time-based clearing on the next idle gap. The pinned edits are also cleared during full compaction's post-compact cleanup.

The system also captures the baseline `cache_deleted_input_tokens` from the last assistant message. This baseline is needed by the cache break detection system — without it, the token drop from cached edits would trigger a false "cache break" warning.

### Compactable Tools

Not all tool results are safe to clear. The system maintains an allowlist:

- **File reads** — the file can be re-read
- **Shell output** — the output is ephemeral
- **Grep/glob results** — search results can be re-run
- **Web fetch/search** — fetched content can be re-fetched
- **File edits/writes** — the confirmation output is disposable

Tool results from other tools (like user questions, notebook edits, or task management) are NOT cleared — their content may be unreproducible.

### API-Native Context Management

Beyond local microcompact, the system can also request that the API itself manage context. This uses the `context_management` field in API requests to specify edit strategies:

**Tool result clearing**: When input tokens exceed a trigger threshold (default: 180K), the API clears tool results from specific tools (file reads, shell output, grep, glob, web fetches, web searches), keeping the most recent results up to a target token budget (default: 40K). The `clear_at_least` parameter ensures a minimum number of tokens are freed — clearing one small tool result when the context is at 180K wouldn't help.

**Tool use clearing**: A separate strategy for edit/write tools. Rather than clearing their inputs, it excludes their entire tool_use blocks. The distinction matters: for read-like tools, the large output (file content, shell output) is the waste. For write-like tools, the large input (new file content) is the waste.

**Thinking clearing**: For models with extended thinking, old thinking blocks are the largest tokens-per-message contributor. When the user has been idle for over an hour (cache expired anyway), only the last thinking turn is kept. During active use, all thinking turns are preserved.

These strategies compose — multiple edit strategies can be active simultaneously, each targeting a different category of clearable content.

## Tier 2: Full Compact

When microcompact isn't enough — the conversation has genuinely grown past the threshold — the system performs a full compaction. This calls the model to summarize the entire conversation history.

### Pre-Processing

Before the conversation is sent for summarization, two pre-processing steps run:

**Image stripping**: All image and document blocks are removed from user messages, replaced with an `[image]` text marker. Images are large (potentially thousands of tokens each) and not useful for text summarization. The stripping also handles images nested inside tool_result content arrays — a tool might return screenshots that are irrelevant to the summary.

**Attachment stripping**: Skill discovery and skill listing attachments are removed before summarization. These are re-injected post-compact anyway, so including them in the summarization input wastes tokens — the model would summarize content that's about to be restored verbatim.

### The Summarization Prompt

The prompt is the most interesting part. It demands a specific structure:

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

This preamble appears at both the START and END of the prompt (dual-instruction pattern). Why? Models with adaptive thinking sometimes attempt tool calls during summarization despite single instructions. The duplication makes non-compliance less likely.

The prompt then requires nine specific sections in the summary:

1. **Primary Request and Intent** — All explicit user requests and intents, in detail.
2. **Key Technical Concepts** — Important technologies, frameworks, and architectural decisions.
3. **Files and Code Sections** — Every file examined, modified, or created, with full code snippets and rationale.
4. **Errors and Fixes** — Every error encountered and how it was resolved.
5. **Problem Solving** — Problems solved and ongoing troubleshooting approaches.
6. **All User Messages** — ALL non-tool-result user messages. Critical for understanding feedback and corrections.
7. **Pending Tasks** — Explicitly requested tasks that haven't been completed.
8. **Current Work** — Precise detail of work immediately before summarization, with filenames and code snippets.
9. **Optional Next Step** — The next step in line with recent requests, with direct quotes showing task status.

Section 6 ("All user messages") is the most unusual. Summarization typically abstracts away individual messages. But user messages contain corrections ("no, I meant X"), preferences ("always use bun"), and implicit context that a summary might smooth over. Preserving them verbatim prevents the model from drifting away from what the user actually said.

Section 9 requires "direct quotes" from the conversation to justify the suggested next step. This prevents task drift — without quotes, the model might hallucinate a next step that wasn't actually in progress.

### The Analysis Scratchpad

The prompt asks for TWO blocks: `<analysis>` then `<summary>`. The analysis block is a drafting scratchpad — the model walks through the conversation chronologically, identifying requests, decisions, code changes, and errors. This structured thinking improves the quality of the summary that follows.

But the analysis block is stripped before delivery. `formatCompactSummary` removes everything between `<analysis>` tags, extracts the `<summary>` content, and replaces the tags with a "Summary:" header. The user never sees the scratchpad. It exists purely to improve the summary via chain-of-thought reasoning.

### Prompt Cache Sharing

The summarization call sends the entire conversation as context. Normally this means re-tokenizing everything — expensive. But the main conversation's prompt prefix (system prompt, tools, early messages) is already cached from the most recent API call.

The system uses a "forked agent" to reuse this cache. The fork inherits the main conversation's cached parameters (system prompt, tool definitions, user context) and sends them as identical cache-key parameters, so the summarization call gets a cache hit on the shared prefix. The remaining messages (the ones being summarized) are the only new tokens.

A critical constraint: the fork must NOT set `maxOutputTokens`. Setting it would clamp the thinking budget via a formula in the API client, creating a thinking config mismatch that invalidates the cache key. The forked agent uses the model's default output limit. Since compaction is capped at one turn (`maxTurns: 1`), the output naturally stays within bounds.

The fork also skips writing to the prompt cache (`skipCacheWrite: true`) — its response is ephemeral and caching it would waste cache creation tokens. The fork's tool permissions are locked to deny-all (`createCompactCanUseTool`), ensuring the model produces only text, never tool calls.

If the fork fails, the system falls back to a direct streaming call with the compact-specific output cap (20K tokens). Telemetry tracks the cache hit rate to monitor effectiveness — a 98% miss rate in the fork path would cost ~0.76% of fleet-wide cache creation, concentrated in ephemeral environments with cold caches.

During the summarization call, the system sends keep-alive signals every 30 seconds — a session activity signal plus a "compacting" status update. This prevents WebSocket timeouts in IDE integrations where the compaction call might take 30-60 seconds for large conversations.

### Hooks

The compaction system fires four hook events that users can subscribe to:

- **PreCompact** — runs before summarization. Returns optional custom instructions that are merged with the user's instructions. User instructions come first, hook instructions appended.
- **PostCompact** — runs after compaction completes.
- **SessionStart** — runs after compaction to re-trigger initialization logic (CLAUDE.md reload, etc.).

These hooks allow plugins and IDE integrations to inject context, clear their own caches, or perform cleanup. Hook results are included in the post-compact message array as `hookResults`.

### Three Prompt Variants

The system has three compaction modes:

- **Full compact**: Summarize the entire conversation. Used by auto-compact and `/compact`.
- **Partial compact ("from")**: Summarize only messages after a selected point, preserving earlier messages. Preserves the prompt cache (early messages stay).
- **Partial compact ("up_to")**: Summarize messages before a selected point, keeping later messages. Invalidates the prompt cache (the kept messages move to the end).

The "from" variant adds: "Earlier retained messages are NOT re-summarized." The "up_to" variant changes section 8 from "Current Work" to "Work Completed" and adds "Context for Continuing Work" — since newer messages follow the summary, the summary needs to set up context rather than continue work.

## The Prompt-Too-Long Retry Loop

Sometimes the conversation is so large that the compaction request itself exceeds the context window. The system handles this with a retry loop:

```
for attempt in 1..MAX_PTL_RETRIES (3):
  try:
    stream summary
    return result
  catch PromptTooLong:
    messages = truncateHeadForPTLRetry(messages, errorResponse)
    if messages is null:
      throw "Conversation too long. Press esc twice to go up a few messages and try again."
```

`truncateHeadForPTLRetry` groups messages by API round (one group per model response with its tool results). It calculates how many tokens to drop based on the error response's token gap. If the gap is unparseable (some Vertex/Bedrock error formats), it falls back to dropping 20% of groups. It drops the oldest groups first.

A subtle self-referential bug was fixed: the function strips its own synthetic marker from a previous retry before grouping. Otherwise the marker becomes its own group at index 0, and the 20% fallback stalls — it drops only the marker, re-adds it on the next retry, and makes zero progress. The fix checks if the first message is the marker (by content match and `isMeta` flag) and strips it before grouping.

If the truncated messages would start with an assistant message (violating the API's alternation requirement), a synthetic user message is prepended: `[earlier conversation truncated for compaction retry]`.

If ALL groups would need to be dropped (nothing left to summarize), the function returns null and the user sees an error message suggesting they press Escape to go back a few messages.

## Post-Compact: What Survives the Boundary

After summarization, the old messages are replaced wholesale. The new message array is:

```
[
  boundaryMarker,        // System message marking the compaction point
  ...summaryMessages,    // The formatted summary as a user message
  ...messagesToKeep,     // Preserved messages (partial compact only)
  ...attachments,        // Re-injected context
  ...hookResults,        // User hook output
]
```

### The Boundary Marker

A system message that records metadata about the compaction:

- **trigger**: "manual" (user ran `/compact`) or "auto" (threshold exceeded)
- **preTokens**: token count before compaction (for analytics)
- **messagesSummarized**: how many messages were replaced
- **logicalParentUuid**: UUID of the last pre-compact message (enables fork/rewind to find the original conversation)
- **preCompactDiscoveredTools**: tool names seen before compaction (for re-announcing)
- **preservedSegment**: head/anchor/tail UUIDs (for partial compact message relinking)

The boundary is the anchor point. Everything before it is gone (replaced by the summary). Everything after it is the new conversation.

### Cache Break Detection

After compaction, the prompt cache baseline is stale. The token count drops legitimately — old messages were replaced with a shorter summary. Without intervention, the cache break detection system would see the drop in `cache_read_input_tokens` and flag a "cache break" warning.

The fix: `notifyCompaction()` resets the previous cache read baseline to null. The next API call establishes a fresh baseline. The detection system compares subsequent calls against this new baseline, ignoring the compaction-induced drop.

The cache break detector itself uses dual thresholds: a drop must be both >5% of the previous cache read AND >2,000 tokens to be flagged. Small fluctuations from server-side cache management are ignored.

### Re-Injected Attachments

The system generates attachments in parallel to restore context that the summary might have compressed too aggressively:

**Recently-read files** — The 5 most recently accessed files are re-read with fresh content (not cached — the file may have changed since it was first read). Each file is capped at 5,000 tokens, with a total budget of 50,000 tokens. Plan files and memory files (CLAUDE.md) are excluded — they have their own injection paths via the system prompt.

The file selection uses recency ordering from the file read state tracker. Files already present in preserved messages (partial compact) are skipped to avoid duplication. The deduplication scans preserved messages for Read tool_use blocks and collects their file paths. It also skips files that had the "FILE_UNCHANGED" stub (a deduplication marker that points at an earlier full read of the same file).

Each file is re-read via the actual File Read tool at restoration time. This means the restored content reflects the file's CURRENT state, not its state when it was first read. If the model edited a file 30 turns ago and the file was later modified by other tools, the post-compact restoration shows the latest version.

**Active skills** — Skills invoked during the session are preserved, sorted most-recent-first. Each skill is capped at 5,000 tokens (truncated with a marker telling the model it can re-read the full content). Total budget: 25,000 tokens.

**Plan file** — If a plan exists for the current session, it's re-injected as an attachment.

**Plan mode** — If the user is currently in plan mode, an attachment ensures the model continues in plan mode after compaction.

**Async agent status** — Background agents that are still running or recently finished get status attachments. This prevents the model from spawning duplicate agents after losing the original creation context.

**Tool deltas** — The full tool set is re-announced. After compaction, the model needs to know what tools are available — the original tool announcements from earlier in the conversation are gone.

**MCP instructions** — Model Context Protocol tool instructions are re-injected for any MCP servers with deferred tool loading.

### Post-Compact Cleanup

After compaction, 10+ caches are cleared because their contents reference pre-compact state:

- Microcompact tracking state (tool IDs no longer valid)
- User context cache (forces CLAUDE.md reload and InstructionsLoaded hook)
- Memory file cache (allows fresh memory file detection)
- System prompt sections (may reference pre-compact state)
- Classifier approvals (permissions may have changed)
- Bash permission speculative checks (stale command analysis)
- Session messages cache (old messages gone)
- Beta tracing state
- File content cache (for commit attribution)

The cleanup is careful about main-thread vs. subagent scope. Subagents run in the same process and share module-level state with the main thread. Clearing state during a subagent compaction would corrupt the main thread. The cleanup checks the query source prefix (`repl_main_thread` or `sdk`) before resetting shared state.

One deliberate non-clear: the set of sent skill names. Re-injecting the full skill listing post-compact costs ~4,000 tokens of pure cache creation. The model still has the skill tool in its schema, and the invoked_skills attachment preserves content for used skills. Skipping re-injection saves tokens on every compaction.

## Auto-Compact Orchestration

The auto-compact flow ties everything together:

```
function autoCompactIfNeeded(messages):
  if consecutiveFailures >= 3 → return (circuit breaker)
  if not shouldAutoCompact(messages) → return

  // Try session memory compaction first (cheap, no model call)
  result = trySessionMemoryCompaction(messages)
  if result:
    cleanup, return success

  // Fall back to full compaction (expensive, model call)
  result = compactConversation(messages,
    suppressFollowUpQuestions = true,
    isAutoCompact = true)
  if result:
    cleanup, reset failures to 0, return success

  // Failure: increment circuit breaker
  consecutiveFailures++
  if consecutiveFailures >= 3:
    log "circuit breaker tripped"
```

When auto-compact triggers, it suppresses follow-up questions. The model receives: "Continue without asking user further questions. Resume directly — do not acknowledge summary, do not recap, do not preface. Pick up last task as if break never happened." This prevents the jarring experience of the model suddenly asking "Would you like me to continue?" mid-task.

In autonomous/proactive mode, the continuation message is even stronger: "You are running in autonomous mode. This is NOT first wake-up. Continue work loop — pick up where you left off. Do not greet or ask what to work on."

For manual `/compact`, the user can provide custom instructions (e.g., "focus on the authentication work") that are appended to the summarization prompt.

### Recompaction Tracking

The system tracks compaction chains — situations where auto-compact fires, the conversation grows past the threshold again, and auto-compact fires a second time. Each compaction records:

- Whether this is a recompaction in a chain
- Turns since the previous compaction
- The previous compaction's turn ID
- The auto-compact threshold that triggered it
- The query source that was active when triggered

This metadata feeds into telemetry for monitoring compaction quality. If compaction produces summaries that are too verbose (consuming too many tokens), the conversation will recompact quickly — a signal that the summarization prompt needs tuning.

## Tier 3: Session Memory Compact

Full compaction is expensive — it sends the entire conversation to the model and waits for a summary. Session memory compaction is an experimental alternative that skips the model call entirely.

### How Session Memory Works

Throughout the conversation, a background process periodically extracts "session memory" — a structured markdown file with sections like Current State, Task Specification, Files and Functions, Errors & Corrections, and a Worklog.

The extraction triggers based on two conditions:

```
trigger = (tokenGrowth >= minimumTokensBetweenUpdate)
          AND (toolCalls >= toolCallsBetweenUpdates OR noToolCallsInLastTurn)
```

The extraction runs in a forked subagent — isolated from the main conversation, using the API's cache-safe parameters to avoid polluting the main prompt cache. The forked agent can ONLY use the file edit tool, and only on the session memory file. It reads the current notes, the recent conversation, and updates the file.

Section sizes are enforced: 2,000 tokens per section, 12,000 tokens total. If a section exceeds its limit, the extraction prompt includes a reminder to condense. This prevents the session memory file from growing without bound.

### Using Session Memory for Compaction

When auto-compact triggers, it tries session memory compaction first:

```
function trySessionMemoryCompaction(messages):
  if feature disabled → null
  if no session memory file → null
  if session memory is empty template → null

  wait for any in-progress extraction to complete

  calculate which messages to keep (most recent, meeting minimum thresholds)
  adjust keep-index to preserve API invariants (tool_use/result pairs, thinking blocks)

  create compaction result using session memory as the summary
  estimate post-compact token count

  if postCompactTokens >= autoCompactThreshold → null  // Would immediately re-trigger
  return result
```

The "messages to keep" calculation balances recency against token budget:

```
start from first unsummarized message
if already at maxTokens (40K): stop
if already meeting minTokens (10K) AND minTextBlockMessages (5): stop
otherwise: expand backward until one of above conditions met
floor: most recent compact boundary (can't go before it)
```

The API invariant adjustment ensures the keep boundary doesn't split tool_use/tool_result pairs or thinking blocks that share the same message ID. It walks backward to include any orphaned pairs.

The token count estimate guards against a pathological loop: if the post-compact token count would already exceed the auto-compact threshold, the system rejects the result and returns null. Without this guard, session memory compaction would succeed, the next turn would trigger auto-compact again (because the kept messages are too large), triggering another session memory compact, and so on.

Session memory compaction is significantly cheaper — no model call, no 20K output token generation. But it depends on the quality of the pre-extracted notes, which may miss nuances that a dedicated summarization call would capture.

### The Session Memory File Format

The extraction prompt defines a structured markdown template with ten sections:

- **Session Title** — 5-10 word title
- **Current State** — pending tasks, next steps
- **Task Specification** — what the user asked, design decisions
- **Files and Functions** — important files and why they're relevant
- **Workflow** — bash commands, execution order, interpreting output
- **Errors & Corrections** — encountered errors and their fixes
- **Codebase and System Documentation** — important components, how they fit together
- **Learnings** — what worked, what to avoid
- **Key Results** — exact user-requested output
- **Worklog** — step-by-step summary of work done

Each section is capped at 2,000 tokens. The total file is capped at 12,000 tokens. When a section grows past its limit, the extraction prompt includes a reminder: "section must be condensed." When the total exceeds 12,000: "CRITICAL: file exceeds max, aggressively shorten."

Before including session memory in a compaction result, the content is further truncated via `truncateSessionMemoryForCompact`. This truncates each section to ~2,000 tokens (8,000 characters), preserving section headers and italic descriptions. An overflow marker tells the model it can read the full file if needed.

### The Fallback Chain

The full compaction fallback chain is:

1. **Session memory compact** — cheapest, fastest, depends on extraction quality
2. **Full compact with prompt cache sharing** — expensive but thorough, reuses cached prefix
3. **Full compact streaming** — fallback if cache sharing fails
4. **PTL retry with head truncation** — if compact itself exceeds context window
5. **User error message** — "Press esc twice to go up a few messages and try again"

Each tier is tried only when the previous one fails or is unavailable.

## The Cost Model

| Stage | Input Tokens | Output Tokens | Latency |
|-------|-------------|---------------|---------|
| Microcompact (cached) | 0 | 0 | ~0 |
| Microcompact (time-gap) | 0 | 0 | ~0 |
| Session memory compact | 0 | 0 | ~0 |
| Full compact | ~167K | up to 20K | 1 model turn |

For full compact at the 200K threshold: 167K tokens of old history become ~20K tokens of summary plus rehydrated attachments. Net savings: ~147K tokens. The cost is one model turn's latency plus the input/output token charges for the summarization call.

Microcompact and session memory compact are essentially free — no model call, no token charges. They exist to defer the expensive full compact as long as possible.

## The Full Round-Trip

To understand how the pieces fit together, trace one complete auto-compact cycle through the system:

**1. The REPL starts the query loop.** When the user sends a message, `REPL.tsx` calls the `query()` generator, which yields messages as they arrive. The REPL consumes them via `for await (event of query(...))` and appends each to the UI.

**2. Microcompact runs first.** Before anything else in the query loop, `microcompactMessages` checks whether tool results should be cleared. If the cache is warm, it queues cache edits. If the user was idle for an hour, it mutates the message array directly.

**3. Auto-compact checks the threshold.** `autoCompactIfNeeded` is called with the current messages, the tool use context, cache-safe parameters, and the tracking state. The tracking state is a persistent object threaded through the query loop — it carries the circuit breaker count, the turn counter, and the previous compact's turn ID across iterations.

**4. The compaction runs.** If the threshold is exceeded, the system tries session memory first, then falls back to full compact. The full compact spawns a forked agent with the summarization prompt, streams the response, handles PTL retries if needed, and builds the post-compact message array.

**5. Post-compact messages are yielded.** The query generator yields the boundary marker, summary messages, attachments, and hook results one at a time. Each `yield` sends the message back to the REPL.

**6. The REPL detects the boundary.** When `onQueryEvent` receives a compact boundary message, it handles it specially: in fullscreen mode, it keeps pre-compact messages for scrollback. In normal mode, it replaces the entire message array with just the boundary. It bumps the conversation ID (a random UUID), which forces React to remount all message rows — ensuring stale UI state doesn't persist.

**7. The query loop continues.** After yielding post-compact messages, the query loop replaces its internal `messagesForQuery` with the compacted set and continues to the API call. The model sees only the summary, attachments, and the new user message. The tracking state is reset: turn counter to 0, turn ID to a fresh UUID, consecutive failures to 0.

**8. If the API call fails with prompt-too-long**, reactive compaction (when enabled) catches it. Reactive compact is the mirror of proactive auto-compact — instead of preventing the PTL error, it recovers from one. The error is "withheld" (not yielded to the REPL) while recovery is attempted. If recovery succeeds, the query loop continues with the compacted messages. If it fails, the withheld error is yielded and the session returns to the user.

This round-trip — REPL → query generator → microcompact → auto-compact → forked agent → stream → boundary → yield → REPL — is the complete execution path. Every compaction, whether manual or auto, follows this flow.

## Closing

Every Claude Code conversation manages its context window through this pipeline:

1. **Token monitoring** — canonical context size measurement, parallel tool call handling, threshold comparison with 13K buffer
2. **Circuit breaker** — max 3 consecutive failures before stopping auto-compact attempts
3. **Microcompact** — clear stale tool results (time-based mutation or cached server-side edits) without a model call
4. **Full compact** — 9-section summarization prompt, analysis scratchpad, NO_TOOLS dual-instruction, PTL retry with head truncation
5. **Prompt cache sharing** — forked agent reuses the main conversation's cached prefix for the summarization call
6. **Post-compact rehydration** — 5 recent files (50K budget), active skills (25K budget), plan files, async agent status, tool deltas, MCP instructions
7. **Post-compact cleanup** — 10+ caches cleared, main-thread/subagent scope isolation, deliberate non-clears for cost savings
8. **Session memory compact** — pre-extracted markdown notes as a cheap alternative to model-based summarization

The system is designed to be invisible. The user keeps working. The conversation keeps going. Behind the scenes, context is compressed, caches are managed, and critical information is preserved. The only visible sign is a brief "Compacted..." message — and even that can be expanded to see the full original transcript.

The fail-closed principle applies here too, but differently than in security. When compaction fails, the system doesn't silently drop messages. It retries with progressively more aggressive truncation, circuit-breaks after repeated failures, and ultimately asks the user to intervene. The alternative — silently losing context — would be worse than any interruption.

The design reflects a hierarchy of priorities: correctness (never lose context silently) over cost (minimize API calls) over latency (minimize user-visible delay). Microcompact optimizes for cost and latency. Full compact prioritizes correctness. Session memory compact tries to get all three. The fallback chain ensures that even in adversarial conditions — massive conversations, API errors, extraction failures — the system degrades gracefully rather than catastrophically.

---

*Follow me on [X](https://x.com/oldeucryptoboi) — I post as @oldeucryptoboi.*
