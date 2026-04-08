---
title: "What Happens When Claude Code Calls the API"
description: "The full client-side pipeline behind every Claude Code API call — request construction, prompt caching, retry strategies, streaming, error recovery, cost tracking, and rate limit management. Defense in depth from source code."
pubDate: 2026-04-07
tags: ["Claude Code", "Anthropic", "API", "Architecture", "Developer Tools"]
---

## The Problem

You type a message. The model needs to see it, along with every previous message, the system prompt, tool schemas, and various configuration. That context gets serialized into an HTTP request, sent to a remote server, and a response streams back as server-sent events. Simple enough — until you consider everything that can go wrong.

The server can be overloaded (529). Your credentials can expire mid-session. The response can be too long for the context window. The connection can go stale. The server can tell you to back off for five minutes, or five hours. The model can try to call a tool that failed three turns ago. Your cache — the thing saving you 90% on input costs — can silently break because a tool schema changed.

The naive approach is: send request, get response, show to user. One function, maybe a try/catch. This fails because a single API call in an agentic loop is not a one-shot operation. It's the inner loop of a system that runs for hours, making hundreds of calls, where each call builds on the state of every previous call. A retry strategy that works for a one-shot chatbot (wait and retry) causes cascading amplification in a capacity crisis. A token counter that's off by 5% will eventually overflow the context window. A cache break you don't detect silently triples your costs.

The design principle is **defense in depth with fail-visible defaults**. Every failure should either be recovered automatically or surfaced to the user with a specific recovery action. Silent failures — where the system degrades without anyone noticing — are the enemy. Cache breaks get detected and logged. Token counts get cross-checked against API-reported usage. Retry decisions consider not just "can we retry" but "should we, given what everyone else is doing right now."

This article traces the full client-side pipeline: request construction, caching, retries, streaming, error recovery, cost tracking, and rate limit management. Everything here is verifiable from the source code. The server side — tokenization, routing, inference, post-processing — is invisible to the client and won't be covered.

## Building the Request

### The System Prompt

Consider what the model needs to know before it sees your message. Its identity, its behavioral rules, what tools it has, how to use them, what tone to take, what language to write in, what project it's working on, what it remembered from previous sessions, what MCP servers are connected. This is the system prompt — a multi-kilobyte payload assembled from ~15 separate section generators.

The prompt has a deliberate physical layout. Everything that stays constant across turns — identity, coding guidelines, tool instructions, style rules — sits at the top. Everything that changes per turn — memory, language preferences, environment info, MCP instructions — sits at the bottom, after an internal boundary marker.

Why this split? The API caches the prompt prefix. On turn 2, the server recognizes the cached prefix and reads it cheaply. If a dynamic section (say, updated memory) sat in the middle, it would invalidate everything after it. By putting all dynamic content at the end, the stable prefix stays cached and only the changing tail incurs write costs.

The system prompt also has a priority hierarchy. An override replaces everything (used by the API parameter). Otherwise: agent-specific prompts (for subagents) > custom prompts (user-specified) > default prompt. An append prompt (from settings like CLAUDE.md) is always added at the end, regardless of which base prompt was selected. This means your CLAUDE.md instructions survive even when the system switches to a subagent prompt.

### Messages

The internal conversation history is a rich format with UUIDs, timestamps, tool metadata, and attachment links. The API expects a simpler format: alternating user/assistant messages with typed content blocks.

Two conversion functions transform the internal format. Both clone their content arrays before modification — a defensive pattern that prevents the API serialization layer from accidentally mutating the in-memory conversation state. This matters because the same message objects get reused across retry attempts and displayed in the UI.

Before conversion, messages pass through a compression pipeline that runs on every API call:

1. **Tool result budgeting** — Caps the total size of tool results per message. A tool that returned 50KB of output gets truncated.
2. **History snipping** — Removes the oldest messages when the conversation exceeds a threshold.
3. **Microcompaction** — Clears stale tool results (file reads, shell output, search results) when the prompt cache has expired and they'll be re-tokenized anyway.
4. **Context collapse** — Applies staged summarization to older conversation segments.
5. **Autocompaction** — Full model-based conversation summary when approaching the context limit.

After conversion, additional cleanup runs:

- **Tool result pairing** — Every `tool_use` block from the model must have a matching `tool_result`. Orphaned tool uses (from aborts, fallbacks, or compaction) get synthetic placeholder results. The API rejects unpaired blocks, and this failure mode is subtle enough that it has dedicated diagnostic logging.
- **Media stripping** — Caps total media items (images, PDFs) at 100 per request. Earlier items are stripped first. This prevents conversations that accumulate many screenshots from exceeding payload limits.

### Prompt Caching

Caching is the most financially significant optimization. On a long session, 90%+ of input tokens may be cache reads. The difference: on a $5/Mtok model, cache reads cost $0.50/Mtok — a 90% discount.

The client places cache markers (`cache_control` directives) at two levels:

- **System prompt blocks**: Every block gets a marker. The server caches them as a unit.
- **Message history**: A single breakpoint at the last message (or second-to-last if skip-write is set). Everything before this point is eligible for caching.

Tool results that appear before the cache breakpoint get `cache_reference` tags linking them to their tool use IDs. This enables server-side cache editing — the server can delete a specific cached tool result without invalidating the entire prefix. This is how the system reclaims space from old tool results while keeping the cache warm.

Cache control details vary by eligibility:

```
type: ephemeral
ttl: 5 minutes (default) or 1 hour (for eligible users)
scope: global (shared across sessions) or unset
```

The 1-hour TTL is gated on subscriber status (not in overage) AND an allowlist of query sources. The allowlist uses prefix matching — `repl_main_thread*` covers all output style variants. This prevents background queries (title generation, suggestions) from claiming expensive 1-hour cache slots.

### Tools, Thinking, and Extra Parameters

Each tool gets serialized to a JSON schema with name, description, and input schema. MCP tools can be deferred — the model sees the tool name but requests full details on demand, reducing the upfront token cost when dozens of MCP tools are connected.

Thinking has three modes. **Adaptive**: the model decides how much to reason (latest models only). **Budget**: a fixed token budget for thinking. **Disabled**: no thinking blocks. When thinking is enabled, the API rejects requests that also set `temperature`, so the client forces temperature to undefined.

The request body also carries: a speed parameter for fast mode (same model, faster inference, higher cost), an effort level, structured output format, task budgets for auto-continuation, feature flag beta headers, and extra body parameters parsed from an environment variable (for enterprise configurations like anti-distillation).

### The Actual Call

```
create(
  parameters + { stream: true },
  options: { abort_signal, headers: { client_request_id: random_uuid } }
).with_response()
```

Always streaming. Always with an abort signal. The `.with_response()` call extracts both the event stream and the raw HTTP response object. The raw response is needed for header inspection — rate limit status, cache metrics, and request IDs all come from response headers, not the stream body.

The client request ID is a UUID generated per call. It exists because timeout errors return no server-side request ID. When a request times out after 10 minutes, this is the only way to correlate the client failure with server-side logs.

## The Client

Before any request fires, a factory function creates the SDK client. The client is provider-specific:

- **Direct API**: API key or OAuth token authentication
- **AWS Bedrock**: AWS credentials (bearer token, IAM, or STS session)
- **Azure Foundry**: Azure AD credentials or API key
- **Google Vertex AI**: Google Application Default Credentials with per-model region routing

All four providers return the same base type, so downstream code doesn't branch on provider. The provider-specific complexity is confined to the factory.

A design trade-off in the Vertex setup: the Google auth library's auto-detection hits the GCE metadata server when no credentials are configured, which hangs for 12 seconds on non-GCE machines. The client checks environment variables and credential file paths first, only falling back to the metadata-server path when neither is present. This trades a longer code path for avoiding a 12-second hang in the common case.

Every request carries session-identifying headers: an app identifier (`cli`), a session ID, the SDK version, and optionally a container ID for remote environments. Custom headers from an environment variable (newline-separated `Name: Value` format) are merged in. For first-party API calls, the SDK's fetch function is wrapped to inject the client request ID and log the request path for debugging.

## Streaming

### What the User Sees

While the API call is in flight, the user sees a spinner with live feedback. The spinner shows the current mode ("Thinking...", "Reading files...", "Running tools..."), an approximate token count updated in real-time as stream chunks arrive, and the elapsed time. If the stream stalls for more than 3 seconds, the spinner changes to indicate the stall visually. If the stall exceeds 30 seconds, the UI offers a contextual tip.

During retries, the user sees a countdown: "Retrying in X seconds..." with the current attempt number and maximum retries. This is the retry generator's yielded status messages being rendered — the async generator architecture means the UI stays responsive even during long backoff waits.

When a rate limit warning is active, the notification bar shows utilization percentage and reset time. When context runs low, a token warning shows remaining capacity and distance to the auto-compact threshold. When a model fallback occurs, a system message appears explaining the switch.

All of this feedback comes from the same event stream — the query loop yields events (stream chunks, retry status, error messages, compaction summaries) and the UI renders them in real-time. Nothing blocks on the complete response.

### The Event Protocol

The response arrives as server-sent events:

```
message_start     → initialize, extract initial usage
content_block_start → begin text / thinking / tool_use block
content_block_delta → accumulate content chunks
content_block_stop  → finalize block
message_delta     → update total usage, set stop reason
message_stop      → end of stream
```

Text deltas are concatenated. Tool use inputs arrive as JSON fragments that are reassembled into a complete JSON object by the final `content_block_stop`. Thinking blocks accumulate both thinking text and a cryptographic signature (for verification).

### The Idle Watchdog

A timer tracks the interval between stream chunks. If no data arrives for 90 seconds, the request is aborted. A warning fires at 45 seconds. This catches a failure mode that TCP timeouts don't: the connection is alive (TCP keepalives succeed) but the server has stopped sending data. Without the watchdog, the client would hang silently for the full 10-minute request timeout.

The 90-second threshold is configurable via environment variable. The trade-off: too short and you abort legitimate long-thinking responses; too long and you waste minutes on hung connections.

### Streaming Tool Execution

When the model emits a tool use block, tool execution can start immediately — while the model might still be generating text or additional tool calls. If the model makes three tool calls and each takes 5 seconds, sequential execution adds 15 seconds. With streaming execution, the first tool starts as soon as it's emitted, and all three may finish by the time the response completes.

If a model fallback occurs mid-stream (3 consecutive overload errors trigger a switch to a fallback model), the streaming executor's pending results are discarded. Tools are re-executed after the fallback response arrives. This prevents stale results from a partially-failed request from contaminating the fallback response.

### Resource Cleanup

When streaming ends — normally, on error, or on abort — the client explicitly releases resources: the SDK stream object is cleaned up, and the HTTP response body is cancelled. This is a defensive pattern against connection pool exhaustion. In a long session with hundreds of tool loops, each API call opens a connection. Without explicit cleanup, idle connections accumulate until the pool is full and new requests fail with connection errors.

### Post-Response Recovery

When the model responds but the response is problematic (no tool calls, but an error condition), the query loop has fallback strategies before surfacing the error:

- **Prompt too long**: First, drain any staged context collapses. If that doesn't free enough space, try reactive compaction — an aggressive, single-shot compression of the conversation. If that also fails, surface the error with a `/compact` hint.
- **Max output tokens hit**: First, try escalating from 8K to 64K output tokens (one-time). If still hitting limits, inject a "Resume directly from where you left off" message and retry. Maximum 3 retries. This handles the case where the model's response is legitimately long (a large code generation) rather than pathologically stuck.
- **Media size errors**: Try reactive compaction with media stripping — removing images and documents that pushed the request over the payload limit.

Each strategy is tried once per error type. The system doesn't loop on recovery.

## The Retry Wrapper

Every API call is wrapped in a retry generator. It yields status messages during waits (so the UI can show "Retrying in X seconds...") and returns the final result on success.

### The Decision Tree

When an error occurs, the handler walks through a priority-ordered sequence:

**User abort** → Throw immediately. No retry.

**Fast mode + rate limit (429) or overload (529)** → Check the retry-after header:
- Under 20 seconds: Wait and retry at fast speed. This preserves the prompt cache — switching speed would change the model identifier and break the cache.
- Over 20 seconds or unknown: Enter a cooldown period (minimum 10 minutes). During cooldown, requests use standard speed. This prevents spending 6x the cost on retries during extended overload.
- If the server signals that overage isn't available (via a specific header), fast mode is permanently disabled for the session.

**Overload (529) from a background source** → Drop immediately. Background work (title generation, suggestions, classifiers) doesn't deserve retries during a capacity crisis. Each retry is 3–10x gateway amplification. The user never sees background failures anyway. New query sources default to no-retry — they must be explicitly added to a foreground allowlist.

**Consecutive 529 counter** → After 3 consecutive overload errors, trigger a model fallback if one is configured. The counter persists across streaming-to-nonstreaming fallback transitions (a streaming 529 pre-seeds the counter for the non-streaming retry loop). Without a fallback model, external users get "Repeated 529 Overloaded errors" and the request fails.

**Authentication errors** → Re-create the entire SDK client. OAuth token expired (401)? Refresh it. OAuth revoked (403 + specific message)? Force re-login. AWS credentials expired? Clear the credential cache. GCP token invalid? Refresh credentials. The retry gets a fresh client with fresh credentials.

**Stale connection (ECONNRESET/EPIPE)** → Disable HTTP keep-alive (behind a feature flag) and reconnect. Keep-alive is normally desirable, but a stale pooled connection that repeatedly resets is worse than the overhead of new connections.

**Context overflow (input + max_tokens > limit)** → Parse the error for exact token counts, calculate available space with a safety buffer, adjust the max_tokens parameter, and retry. A floor of 3,000 tokens prevents the model from having zero room to respond. If thinking is enabled, the adjustment ensures the thinking budget isn't silently eliminated.

**Everything else** → Check if retryable (connection errors, 408, 409, 429, 5xx → yes; 400, 404 → no). Calculate delay. Sleep. Retry.

### Backoff

```
base_delay = min(500ms * 2^(attempt-1), max_delay)
jitter = random() * 0.25 * base_delay
delay = base_delay + jitter
```

The jitter is 0-25% of the base, preventing thundering herd when many clients retry simultaneously. If the server sends a `Retry-After` header, that value overrides the calculated delay.

Three backoff modes exist:

- **Normal**: Up to 10 attempts, max delay grows with attempts.
- **Persistent** (headless/unattended sessions): Retries 429 and 529 indefinitely with a 5-minute cap. Long sleeps are chunked into 30-second intervals, and each chunk yields a status message so the host environment doesn't kill the session for inactivity. A 6-hour absolute cap prevents pathological loops.
- **Rate-limited with reset timestamp**: The server sends an `anthropic-ratelimit-unified-reset` header with the Unix timestamp when the rate limit window resets. The client sleeps until that exact time rather than polling with exponential backoff.

### The x-should-retry Header

The server can explicitly tell the client whether to retry via `x-should-retry: true|false`. But the client doesn't always obey:

- **Subscribers hitting rate limits**: The server says "retry: true" (the limit resets in hours). But the client says no — waiting hours is not useful. Enterprise users are an exception because they typically use pay-as-you-go rather than window-based limits.
- **Internal users on 5xx errors**: The server may say "retry: false" (the error is deterministic). But internal users can ignore this for server errors specifically, because internal infrastructure sometimes returns transient 5xx errors that resolve on retry.
- **Remote environments on 401/403**: Infrastructure-provided JWTs can fail transiently (auth service flap, network hiccup). The server says "don't retry with the same bad key" — but the key isn't bad, the auth service is flapping. So the client retries anyway.

Each of these is a case where the client has context the server doesn't. The server sees "this request failed with status X." The client knows "I'm a subscriber who can't wait 5 hours" or "my auth is infrastructure-managed, not user-provided."

## Error Classification

When retries are exhausted, the error is converted into a user-facing message with a recovery action. Over 20 specific error patterns map to targeted messages:

| Pattern | User Sees | Recovery |
|---------|-----------|----------|
| Context too long with token counts | "Prompt is too long" | `/compact` |
| Model not available | Subscription-aware message | `/model` |
| API key invalid | "Not logged in" | `/login` |
| OAuth revoked | "Token revoked" | `/login` |
| Credits exhausted | "Credit balance too low" | Add credits |
| Rate limit with reset time | Per-plan message | Wait or `/upgrade` |
| PDF exceeds page limit | Size limit shown | Reduce pages |
| Image too large | Dimension limit shown | Resize |
| Bedrock model access denied | Model access guidance | Request access |
| Request timeout | "Request timed out" | Retry |

Messages are context-sensitive. Interactive sessions show keyboard shortcuts ("esc esc" to abort). SDK sessions show generic text. Subscription users get different error messages than API key users. Internal users get a Slack channel link for rapid triage.

Separately, every error gets classified into one of 25 analytics types (`rate_limit`, `prompt_too_long`, `server_overload`, `auth_error`, `ssl_cert_error`, `unknown`, etc.) for aggregate monitoring. This dual classification — human-readable + machine-readable — lets the same error inform both the user and the engineering dashboard.

### The 529 Detection Problem

The SDK sometimes fails to pass the 529 status code during streaming. The server sends 529, but by the time the error reaches the client, the status field may be undefined or different. The client works around this by also checking the error message body for the string `"type":"overloaded_error"`. This string-matching fallback is fragile — if the API changes the error format, it breaks — but it catches a real class of misclassified overload errors that the status code alone misses.

Similarly, the "fast mode not enabled" error is detected by string-matching the error message (`"Fast mode is not enabled"`). The code includes a comment noting this should be replaced with a dedicated response header once the API adds one. String-matching error messages is a known anti-pattern, but when the alternative is failing to detect a recoverable error, fragility is the better trade-off.

## Token Counting and Cost Tracking

### How Tokens Are Counted

The canonical context size function combines two sources:

1. **API-reported usage**: Walk backward through messages to find the last assistant message with a `usage` field. This is the server's authoritative token count at that point.

2. **Client-side estimation**: For messages added after the last API response (the user's new message, any attachment messages), estimate tokens using heuristics: ~4 characters per token for text, 2,000 tokens flat for images, tool name + serialized input length for tool use blocks. Pad by 33%.

The estimation is intentionally conservative. Overestimating triggers compaction too early (wastes a few tokens of capacity). Underestimating triggers a prompt-too-long error (wastes an entire API call).

A subtlety with parallel tool calls: when the model makes N tool calls in one response, streaming emits N separate assistant records sharing the same response ID. The query loop interleaves tool results between them: `[assistant(id=A), tool_result, assistant(id=A), tool_result, ...]`. The token counter must walk back to the FIRST message with the matching ID so all interleaved tool results are included. Stopping at the last one would miss them and undercount.

### Cost Calculation

A per-model pricing table maps model identifiers to rates:

```
sonnet (3.5 through 4.6):  $3 / $15  per million tokens (input/output)
opus 4/4.1:                $15 / $75
opus 4.5/4.6:              $5 / $25
opus 4.6 fast:             $30 / $150
haiku 4.5:                 $1 / $5
```

Cache reads cost 10% of input price. Cache writes cost 125% of input price. The formula:

```
cost = (input / 1M) * input_rate
     + (output / 1M) * output_rate
     + (cache_read / 1M) * cache_read_rate
     + (cache_write / 1M) * cache_write_rate
     + web_searches * $0.01
```

Fast mode pricing is determined by the server, not the client. The API response includes a `speed` field in usage data. If the server processed the request at standard speed despite a fast-mode request (possible during overload), you pay standard rates. The client trusts this field for billing rather than its own request parameter.

Costs are persisted per-session. On session resume, the client checks that the saved session ID matches before restoring — preventing one session's costs from bleeding into another. Unknown models (new model IDs not yet in the table) fall back to the Opus 4.5/4.6 tier and fire an analytics event so the table can be updated.

## Cache Break Detection

A cache break means the server couldn't read the cached prefix and had to re-process all input tokens. On a 100K-token conversation, that's the difference between paying for 5K tokens (cache read) and 100K tokens (full write). Silent cache breaks are an invisible cost multiplier.

The detection system uses two phases:

**Pre-call**: Before each API call, snapshot the state — hashes of the system prompt, tool schemas, cache control config, model name, speed mode, beta headers, effort level, and extra body parameters.

**Post-call**: After the response, compare cache read tokens to the previous call's value. If reads dropped by more than 2,000 tokens and didn't reach 95% of the previous value, flag a cache break.

When a break is detected, the system identifies which snapshot fields changed: model switch, system prompt edit, tool schema addition/removal, speed toggle, beta header change, cache TTL/scope flip. If nothing changed in the snapshot, it infers a time-based cause: over 1 hour since last call (TTL expiry), over 5 minutes (short TTL expiry), or under 5 minutes (server-side eviction).

A unified diff file is written showing the before/after prompt state. With debug mode enabled, this makes cache break investigation straightforward — you can see exactly which tool schema changed or which system prompt section grew.

State is tracked per query source with a cap of 10 tracked sources to prevent unbounded memory growth. Short-lived sources (background speculation, session memory extraction) are excluded from tracking — they don't benefit from cross-call analysis.

## Rate Limits and Early Warnings

After every API response, the client extracts rate limit headers: status (`allowed`, `allowed_warning`, `rejected`), reset timestamp, limit type (`five_hour`, `seven_day`, `seven_day_opus`), overage status, and fallback availability.

### Early Warnings

Before hitting the actual limit, the client warns users who are burning through quota unusually fast:

```
5-hour window:  warn if 90% used but < 72% of time elapsed
7-day window:   warn if 75% used but < 60% of time elapsed
                warn if 50% used but < 35% of time elapsed
                warn if 25% used but < 15% of time elapsed
```

The intuition: if you've used 90% of your 5-hour quota but only 3.6 hours have passed, you're on pace to hit the wall. The preferred method uses a server-sent `surpassed-threshold` header. The client-side time calculation is a fallback.

False positive suppression: warnings are suppressed when utilization is below 70% (prevents spurious alerts right after a rate limit reset). For team/enterprise users with seamless overage rollover, session-limit warnings are skipped entirely — they'll never hit a wall.

### Overage Detection

When `status` changes from `rejected` to `allowed` while `overageStatus` is also `allowed`, the user has silently crossed from subscription quota to overage billing. The client detects this transition and shows a notification: "You're now using extra usage." This matters because overage has its own cost implications.

### Quota Probing

On startup, a test call checks quota status before the first real query: a single-token request to the smallest model. The call uses `.with_response()` to access the raw headers. This lets the UI show rate limit state immediately rather than waiting for the first user message.

## The Full Round-Trip

Putting it all together, here's one API call:

1. **Message preparation**: microcompact, autocompact, context collapse
2. **Request construction**: system prompt blocks with cache markers, converted messages with cache breakpoints and tool result references, tool schemas, thinking config, beta headers, extra body params
3. **Cache state snapshot**: hash system prompt, tools, config
4. **Retry wrapper**: up to 10 attempts with exponential backoff
5. **Client creation**: provider-specific SDK with auth, headers, fetch wrapper
6. **API call**: streaming request with abort signal and client request ID
7. **Stream processing**: event-by-event content accumulation, idle watchdog
8. **Tool execution**: streaming — start tools as they're emitted, before the response completes
9. **Header extraction**: rate limits, cache metrics, request IDs
10. **Cache break analysis**: compare pre/post token ratios
11. **Cost tracking**: per-model pricing, session accumulation, persistence
12. **Error recovery**: 20+ error patterns → specific recovery actions
13. **Query loop**: process tool results, append to history, loop back

Each turn takes 2–30 seconds. A typical session makes 50–200 calls. The retry system makes those calls resilient to transient failures. The caching system makes them affordable. The error classification system makes failures actionable. And the token counter keeps track of exactly how close you are to the edge of the context window.

The alternative to this defense-in-depth approach is simpler code that fails in opaque ways — silent cost overruns, mysterious context overflows, and retries that amplify outages instead of weathering them. Every layer described here exists because the simpler version broke in production.

The key architectural choices:

- **Async generators everywhere**: The query loop, the retry wrapper, and the stream processor are all async generators. This means every layer can yield events to the UI without blocking. A retry wait yields countdown messages. A compaction yields summary events. The UI stays responsive through multi-minute operations.
- **Trust the server's numbers**: Token counts come from API usage fields, not local tokenization. Cache status is inferred from token ratios, not server state. Cost is calculated from server-reported speed mode, not the client's request. The client doesn't have a tokenizer — it uses character-based estimation for new messages and cross-checks against the server's count on every response.
- **Fail visible, not fail silent**: Cache breaks are logged with diffs. Cost anomalies fire analytics events. Rate limit transitions trigger notifications. Unknown models get tracked. The system is designed so that degradation is always observable, even if it's not always preventable.
- **Context over rules**: The retry handler doesn't just ask "is this error retryable?" It asks "is this error retryable for THIS user on THIS provider in THIS mode?" A subscriber hitting 429 is different from an enterprise user hitting 429. A remote environment hitting 401 is different from a local user hitting 401. The same status code gets different treatment depending on context the server can't see.

---

*Follow me on [X](https://x.com/oldeucryptoboi) — I post as @oldeucryptoboi.*
