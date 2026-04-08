---
title: "How Tool Search Defers Tools to Save Tokens"
description: "Claude Code defers MCP tool definitions behind a search layer, cutting tool token costs by 95%. Inside the deferral logic, scoring algorithm, and compaction snapshots."
pubDate: 2026-04-08
tags: ["Claude Code", "Anthropic", "Architecture", "MCP", "AI Agents"]
---

# How Tool Search Defers Tools to Save Tokens

Claude Code can use dozens of built-in tools and an unlimited number of MCP tools. Every tool the model might call needs a definition — a name, description, and JSON schema — sent with each API request. A single MCP tool definition might cost 200–800 tokens. Connect three MCP servers with 50 tools each, and you're burning 60,000 tokens on tool definitions alone. Every turn. Before the model reads a single message.

That's not sustainable. A 200K context window that loses 30% to tool definitions before the conversation starts is a bad experience. The model has less room to think, compaction triggers sooner, and cost per turn climbs.

The naive solution is obvious: don't send tools the model doesn't need. But which tools does the model need? You don't know until it tries to use one. And if the tool definition isn't there when the model tries to call it, the call fails.

Claude Code solves this with a system called **tool search**. When MCP tool definitions exceed a token threshold, most tools are deferred — their definitions are withheld from the API request. In their place, the model gets a single `ToolSearch` tool it can invoke to discover and load tools on demand. The API receives a `tool_reference` content block in the search result, expands it to the full definition, and the model can call the tool on its next turn.

Consider the concrete flow. A user has configured MCP servers for GitHub, Slack, and Jira — 147 tools total. Without tool search, every API call sends 147 tool definitions: ~90,000 tokens. With tool search, the API call sends ~25 built-in tool definitions plus ToolSearch itself: ~15,000 tokens. The model's prompt tells it "147 deferred tools are available — use ToolSearch to load them." When the model needs to create a GitHub issue, it calls `ToolSearch({ query: "github create issue" })`. The system returns a `tool_reference` for `mcp__github__create_issue`. On the next turn, that tool's full schema is available, and the model calls it normally. Total overhead for this discovery: one extra turn, ~200 tokens. Savings over a 20-turn conversation: ~1.5 million tokens.

This article traces the entire pipeline: the deferral decision, the threshold calculation, the search algorithm, the discovery loop across turns, and the snapshot mechanism that preserves discovered tools across context compaction. Every layer is designed around the same principle: **fail closed, fail toward asking**. If anything is uncertain — an unknown model, a proxy gateway, a missing token count — the system falls back to loading all tools, never to silently hiding them.

---

## The Deferral Decision

Not every tool can be deferred. The model needs certain tools on turn one, before it has a chance to search for anything. The deferral decision is a priority-ordered checklist:

```
function isDeferredTool(tool):
    # Explicit opt-out: MCP tools can declare they must always load
    if tool.alwaysLoad is true:
        return false

    # MCP tools are deferred by default (workflow-specific, often numerous)
    if tool.isMcp is true:
        return true

    # ToolSearch itself is never deferred — it's the bootstrap
    if tool.name is "ToolSearch":
        return false

    # Core communication tools are never deferred
    # (Agent, Brief — model needs these immediately)
    if tool is a critical communication channel:
        return false

    # Everything else: defer only if explicitly marked
    return tool.shouldDefer is true
```

The `alwaysLoad` opt-out is the escape hatch. An MCP server can set `_meta['anthropic/alwaysLoad']` on a tool to force it into every API request regardless of deferral mode. This handles tools like a primary database query tool that the model will need on nearly every turn.

Notice the ordering. `alwaysLoad` is checked before the MCP check. This means an MCP tool can opt out of deferral even though MCP tools are deferred by default. And `ToolSearch` is checked after the MCP check, which means if someone wraps ToolSearch in an MCP server (don't), it still won't be deferred. The checklist is a priority chain where each rule can only override the ones below it.

The `shouldDefer` flag at the bottom is for built-in tools that want to participate in deferral without being MCP tools. Currently this isn't widely used, but it exists as an extension point — a built-in tool could mark itself as deferrable if it's rarely needed and expensive to describe.

### Three Modes

The deferral system operates in one of three modes, controlled by an environment variable:

```
function getToolSearchMode():
    # Kill switch: if all beta features are disabled, never defer
    if DISABLE_EXPERIMENTAL_BETAS:
        return "standard"

    value = env.ENABLE_TOOL_SEARCH

    # Explicit "always defer" mode
    if value is truthy or value is "auto:0":
        return "tst"

    # Threshold-based: only defer when tools exceed a token budget
    if value is "auto" or value is "auto:N" where 1 <= N <= 99:
        return "tst-auto"

    # Explicit disable
    if value is falsy:
        return "standard"

    # Default: always defer MCP and shouldDefer tools
    return "tst"
```

The default mode is `tst` — always defer. This is the right default because any user with MCP tools has already accepted the latency of an extra search turn in exchange for a larger effective context window. The `tst-auto` mode provides a middle ground: defer only when the token cost actually justifies it.

### The Threshold Calculation

In `tst-auto` mode, the system measures how many tokens the deferred tools would consume and compares against a budget:

```
threshold = floor(contextWindow * percentage / 100)
# Default percentage: 10%
# For a 200K context model: threshold = 20,000 tokens
```

The token count comes from the API's `countTokens` endpoint when available. The system serializes each deferred tool into its API schema (name + description + JSON schema), sends them to the counting endpoint, and caches the result keyed by the tool name set. The cache invalidates when MCP servers connect or disconnect, changing the tool pool.

There's a subtlety in the counting. The API adds a fixed preamble (~500 tokens) whenever tools are present in a request. When counting tools individually, each count includes this overhead, so counting N tools individually would report N × 500 tokens of phantom overhead. The system subtracts this constant from the total:

```
rawCount = countTokensViaAPI(deferredToolSchemas)
adjustedCount = max(0, rawCount - 500)
```

When the token counting API is unavailable — perhaps the provider doesn't support it, or the network request fails — the system falls back to a character-based heuristic. It sums the character lengths of each tool's name, description, and serialized input schema, then converts using a ratio of 2.5 characters per token:

```
charThreshold = floor(tokenThreshold * 2.5)
totalChars = sum(tool.name.length + tool.description.length + tool.schema.length
                 for each deferred tool)
enabled = totalChars >= charThreshold
```

This heuristic is intentionally conservative. Tool definitions are schema-heavy (lots of short keys and structural characters), which tokenize at a higher density than natural language. A 2.5 chars/token ratio slightly overestimates the token count, biasing toward enabling deferral — the safe direction.

---

## The Search Mechanism

When tool search is enabled, the model sees a `ToolSearch` tool in its tool list. The tool accepts a query string and returns up to 5 results (configurable). There are two query modes.

### Direct Selection

The model can request specific tools by name:

```
ToolSearch({ query: "select:mcp__github__create_issue" })
ToolSearch({ query: "select:Read,Edit,Grep" })  # comma-separated
```

Direct selection is a lookup, not a search. For each requested name, the system checks the deferred tool pool first, then falls back to the full tool set. Finding a tool in the full set that isn't deferred is a no-op — the tool is already loaded — but returning it prevents the model from retrying in a loop.

Why does the fallback to the full tool set matter? After context compaction or in subagent conversations, the model sometimes tries to "select" a tool it previously used, not realizing the tool is already loaded (because its earlier search result was summarized away). Without the full-set fallback, the select would fail, the model would get "no matching deferred tools found," and it would waste a turn figuring out the tool is already available. The fallback makes this a silent success.

### Keyword Search

When the model doesn't know the exact tool name, it searches by keyword:

```
ToolSearch({ query: "slack send message" })
ToolSearch({ query: "+github pull request" })  # + requires term
```

The search algorithm scores each deferred tool against the query terms:

```
function scoreToolForQuery(tool, terms):
    parts = parseToolName(tool.name)
    # "mcp__slack__send_message" -> parts: ["slack", "send", "message"]
    # "NotebookEdit" -> parts: ["notebook", "edit"]

    score = 0
    for term in terms:
        # Exact part match (highest signal)
        if term in parts:
            score += 12 if tool.isMcp else 10

        # Substring match within a part
        elif any(term in part for part in parts):
            score += 6 if tool.isMcp else 5

        # Full name fallback
        elif term in fullName and score is 0:
            score += 3

        # searchHint match (curated capability phrase)
        if wordBoundaryMatch(term, tool.searchHint):
            score += 4

        # Description match (lowest signal, most noise)
        if wordBoundaryMatch(term, tool.description):
            score += 2

    return score
```

MCP tools get slightly higher weight on exact matches (12 vs 10) and substring matches (6 vs 5). This is deliberate: when tool search is active, most deferred tools are MCP tools. Boosting their scores ensures they rank above built-in tools that happen to share terminology.

The `searchHint` field is a curated string that tools can provide to improve discoverability. It's weighted above description matches (4 vs 2) because it's intentional signal — a tool author explicitly saying "this tool handles X" — rather than incidental keyword overlap in a long description.

Description matching uses word-boundary regex (`\bterm\b`) to avoid false positives. Without boundaries, a search for "read" would match every tool whose description contains "already", "thread", or "spreadsheet".

There's also a required-term mechanism. Prefixing a term with `+` makes it mandatory: only tools matching ALL required terms in their name, description, or search hint are scored. This lets the model narrow results when a server has many tools: `+slack send` finds tools with "slack" in the name AND ranks them by "send" relevance.

### A Concrete Scoring Example

Suppose the deferred pool contains these tools:

```
mcp__slack__send_message        (MCP)
mcp__slack__list_channels       (MCP)
mcp__github__create_issue       (MCP)
mcp__email__send_email          (MCP)
```

The model searches: `ToolSearch({ query: "slack send" })`. Here's the scoring:

```
mcp__slack__send_message:
  parts = ["slack", "send", "message"]
  "slack": exact part match, MCP → +12
  "send":  exact part match, MCP → +12
  Total: 24

mcp__slack__list_channels:
  parts = ["slack", "list", "channels"]
  "slack": exact part match, MCP → +12
  "send":  no match in parts, no match in name → +0
  Total: 12

mcp__email__send_email:
  parts = ["email", "send", "email"]
  "slack": no match → +0
  "send":  exact part match, MCP → +12
  Total: 12

mcp__github__create_issue:
  parts = ["github", "create", "issue"]
  "slack": no match → +0
  "send":  no match → +0
  Total: 0
```

Result: `["mcp__slack__send_message", "mcp__slack__list_channels", "mcp__email__send_email"]`. The Slack send tool wins, the other Slack tool ties with the email send tool, and the GitHub tool is excluded. Note how multi-term queries naturally boost tools that match on multiple dimensions — a tool matching both "slack" AND "send" scores 24, while one matching only "slack" scores 12.

The regex patterns are pre-compiled once per search to avoid creating them inside the hot loop (N tools × M terms × 2 checks). Each unique term gets one compiled regex, and all tools share them.

### The MCP Prefix Fast Path

When the query starts with `mcp__`, the system checks for prefix matches before falling through to keyword search:

```
if query starts with "mcp__":
    matches = tools where name starts with query
    if matches found:
        return first maxResults matches
```

This handles the common pattern where the model knows the server name but not the specific action. Searching `mcp__github` returns all GitHub MCP tools without keyword scoring.

### What Search Returns

The search doesn't return tool definitions. It returns `tool_reference` content blocks:

```
# Tool result sent back to the API:
{
    type: "tool_result",
    tool_use_id: "...",
    content: [
        { type: "tool_reference", tool_name: "mcp__github__create_issue" },
        { type: "tool_reference", tool_name: "mcp__github__list_issues" }
    ]
}
```

This is a beta API feature. The API server receives the `tool_reference` block and expands it into the full tool definition in the model's context. The client never sends the definition itself — the API resolves the reference from the deferred schemas that were sent with `defer_loading: true`.

This is the key insight of the architecture. The client marks deferred tools with `defer_loading: true` in their schema, telling the API "here's the definition, but don't show it to the model unless referenced." The `tool_reference` block is the trigger that expands a deferred definition. The model sees the full schema in its context only after a successful search.

Why not just return the full tool definition in the search result? Two reasons. First, the API handles the injection into the model's tool context — the client doesn't need to construct a new API request with the tool added. Second, `tool_reference` is a structured content block that the API validates against the known deferred schemas. The client can't fabricate a tool definition in a tool_result and have it treated as a callable tool. The API is the authority on which tools exist.

### The Two-Layer Gate

For tool search to actually engage, two checks must pass:

**Optimistic check** (fast, stateless): Can tool search possibly be enabled? This runs early — during tool pool assembly — to decide whether ToolSearch itself should be included in the tool list. It checks mode and proxy gateway, but NOT model or threshold. This is called "optimistic" because it says "yes" even if the definitive check might say "no" later.

**Definitive check** (async, contextual): Should tool search be used for this specific API request? This runs at request time with the full context: model name, tool list, token counts. It checks model support, ToolSearch availability, and (for `tst-auto`) the threshold.

The two-layer design avoids a chicken-and-egg problem. You can't check the definitive gate until you've assembled the tool pool. But the tool pool includes ToolSearch. If ToolSearch isn't in the pool, the definitive check will say "ToolSearch unavailable, disable." So the optimistic check decides whether to include ToolSearch, and the definitive check decides whether to use it.

---

## The Discovery Loop

Tool search creates a multi-turn protocol. On turn 1, the model sees only non-deferred tools plus ToolSearch. It calls ToolSearch. On turn 2, the discovered tools are available. But how does the system know which tools to include on turn 2?

### Scanning Message History

Before each API request, the system scans the conversation history for `tool_reference` blocks:

```
function extractDiscoveredToolNames(messages):
    discovered = empty set

    for message in messages:
        # Compact boundaries carry a snapshot (explained later)
        if message is compact_boundary:
            for name in message.metadata.preCompactDiscoveredTools:
                discovered.add(name)
            continue

        # tool_reference blocks only appear in user messages
        # (tool_result is a user-role message in the API)
        if message is not user:
            continue

        for block in message.content:
            if block is tool_result with array content:
                for item in block.content:
                    if item.type is "tool_reference":
                        discovered.add(item.tool_name)

    return discovered
```

The extracted set determines which deferred tools to include in the next request:

```
function filterToolsForRequest(tools, deferredToolNames, discoveredToolNames):
    return tools where:
        # Always include non-deferred tools
        tool.name not in deferredToolNames
        # Always include ToolSearch itself
        OR tool.name is "ToolSearch"
        # Include deferred tools that have been discovered
        OR tool.name in discoveredToolNames
```

This creates an accumulating set. Once a tool is discovered via search, it stays available for the rest of the conversation. The model never needs to re-search for a tool it's already found.

There's an important detail in what gets sent to `toolToAPISchema`. The filtering controls which tools appear in the API's tool array. But the ToolSearch prompt — which lists available deferred tools for the model to see — is generated from the *full* tool list, not the filtered one. This separation ensures the model can always search the complete pool, even though only discovered tools have their schemas sent.

### A Complete Round-Trip

Let's trace a single discovery end-to-end:

**Turn 1**: User says "Create a GitHub issue for this bug."
- System computes deferred set: 147 MCP tools.
- System scans history: no `tool_reference` blocks yet.
- Filtered tools: 25 built-in + ToolSearch. 147 deferred sent with `defer_loading: true`.
- Model sees 26 tools. It knows it needs GitHub. It calls ToolSearch.

**Turn 1 response**: Model generates `tool_use` for ToolSearch with query `"select:mcp__github__create_issue"`.

**Turn 1 result**: System looks up the name, finds it in deferred pool. Returns:
```
{ type: "tool_result", content: [
    { type: "tool_reference", tool_name: "mcp__github__create_issue" }
]}
```

**Turn 2**: System prepares next API request.
- Scans history: finds `tool_reference` for `mcp__github__create_issue`.
- Filtered tools: 25 built-in + ToolSearch + `mcp__github__create_issue`.
- Model sees 27 tools. `mcp__github__create_issue` has full schema. Model calls it.

The discovery cost was one turn and ~200 tokens. The savings persist for the rest of the conversation — `mcp__github__create_issue` stays in the filtered set.

### The defer_loading Flag

Each tool that passes through the filter gets serialized into an API schema. Deferred tools — those in `deferredToolNames` — get an extra field:

```
schema = {
    name: tool.name,
    description: tool.description,
    input_schema: tool.jsonSchema,
    defer_loading: true   # <-- tells API: definition present but hidden
}
```

The `defer_loading` flag is the contract between client and API. It says: "I'm sending you this tool's definition so you can expand `tool_reference` blocks that name it, but don't include it in the model's visible tool list unless a reference triggers it."

Without this flag, sending a tool definition means the model sees it. With it, the definition is staged but invisible until referenced. This is what makes the system work — the API has all the information it needs to expand references without the client re-sending definitions on demand.

### Announcing Available Tools

The model needs to know what tools it can search for. There are two mechanisms for this, gated by a feature flag:

**Pre-message approach** (original): Before the conversation messages, the system prepends a user message listing all deferred tool names:

```
<available-deferred-tools>
mcp__github__create_issue
mcp__github__list_issues
mcp__slack__send_message
...
</available-deferred-tools>
```

**Delta attachment approach** (newer): Instead of prepending the full list every turn, the system computes a diff against what's already been announced:

```
function getDeferredToolsDelta(tools, messages):
    # Scan prior attachment messages for previous announcements
    announced = empty set
    for message in messages:
        if message is attachment of type "deferred_tools_delta":
            for name in message.addedNames: announced.add(name)
            for name in message.removedNames: announced.delete(name)

    deferred = tools where isDeferredTool(tool)
    deferredNames = names of deferred tools
    poolNames = names of all tools

    added = deferred tools not yet announced
    removed = announced tools no longer in the pool AND no longer in base tools
    # Note: a tool that was deferred but is now loaded (undeferred) is NOT
    # reported as removed — it's still available, just loaded differently

    if no changes: return null
    return { addedNames, removedNames }
```

The delta approach has a critical advantage: it doesn't bust the prompt cache. The pre-message approach changes the first message whenever the tool pool changes (MCP server connects late, tools added/removed), which invalidates the cached prefix. Deltas are appended as attachment messages, leaving the prefix stable.

---

## Surviving Compaction

Context compaction summarizes old messages to free space. But compaction destroys `tool_reference` blocks — the summary is plain text, not structured content. If the system can't find tool references after compaction, it thinks no tools have been discovered, and every deferred tool disappears from subsequent requests.

### The Snapshot Mechanism

Before compaction runs, the system takes a snapshot of all discovered tools and stores it on the compact boundary marker:

```
function compact(messages):
    # Snapshot BEFORE summarizing
    discoveredTools = extractDiscoveredToolNames(messages)

    summary = summarize(messages)
    boundaryMarker = createBoundaryMessage(summary)

    if discoveredTools is not empty:
        boundaryMarker.metadata.preCompactDiscoveredTools =
            sorted(discoveredTools)

    return [boundaryMarker, ...remainingMessages]
```

This snapshot appears in three compaction paths: full compaction, partial compaction (which keeps recent messages intact), and session-memory compaction. All three perform the same snapshot.

After compaction, when `extractDiscoveredToolNames` scans the messages, it encounters the compact boundary marker first and reads the snapshot:

```
# Post-compaction message array:
[
    compact_boundary {
        metadata.preCompactDiscoveredTools: ["mcp__github__create_issue", ...]
    },
    ... remaining messages with tool_reference blocks ...
]
```

The scan merges the snapshot with any new references in remaining messages. The union is the full discovered set — nothing is lost.

### Why This Works

The snapshot is idempotent. Multiple compactions each snapshot the accumulated set. If compaction A captures tools {X, Y} and the model later discovers Z, compaction B captures {X, Y, Z}. The set only grows.

Partial compaction scans all messages, not just the ones being summarized. This is deliberate — it's simpler than tracking which tools were referenced in which half, and set union is idempotent, so double-counting is harmless.

---

## Edge Cases and Fail-Closed Design

### Model Support

Not every model supports `tool_reference` content blocks. The system uses a negative list: models are assumed to support tool search **unless** they match a pattern in the unsupported list.

```
UNSUPPORTED_MODEL_PATTERNS = ["haiku"]

function modelSupportsToolReference(model):
    normalized = lowercase(model)
    for pattern in UNSUPPORTED_MODEL_PATTERNS:
        if pattern in normalized:
            return false
    return true   # new models work by default
```

This is a deliberate design choice. A positive list (allowlist) would require code changes for every new model. The negative list means new models inherit tool search support automatically. Only models known to lack the capability are excluded.

The unsupported pattern list can be updated remotely via feature flags, without shipping a new release. This handles the case where a new model launches without `tool_reference` support — the team adds it to the list, and all running instances pick it up.

### Proxy Gateway Detection: A Two-Act Failure

This is a case where a real-world failure, a fix, and a failure of the fix shaped the final design.

**Act 1**: Users routing API calls through third-party proxy gateways (LiteLLM, corporate firewalls) started getting API 400 errors: `"Messages content type tool_reference not supported."` The proxy only accepted standard content types — `text`, `image`, `tool_use`, `tool_result` — and rejected the beta `tool_reference` blocks. Tool search worked fine with direct Anthropic API calls but broke through any intermediary.

**Act 2**: The fix was aggressive: detect non-Anthropic base URLs and disable tool search entirely. This stopped the 400 errors but created a new problem — users with *compatible* proxies (LiteLLM passthrough mode, Cloudflare AI Gateway) lost deferred tool loading. All their MCP tools loaded into the main context window every turn. For users with many MCP tools, this was a significant regression in context efficiency.

The final design balances both failures:

```
function isToolSearchEnabledOptimistic():
    if mode is "standard":
        return false

    # Proxy detection: first-party provider but non-Anthropic URL
    # Only triggers when ENABLE_TOOL_SEARCH is unset (default behavior)
    if ENABLE_TOOL_SEARCH is not set
       AND provider is "firstParty"
       AND baseURL is not a known Anthropic host:
        return false   # proxy would reject tool_reference blocks

    return true
```

The key insight is the `ENABLE_TOOL_SEARCH is not set` condition. When the environment variable is unset, the system assumes unknown proxies can't handle beta features. But setting *any* non-empty value — `true`, `auto`, `auto:10` — tells the system "I know what I'm doing, my proxy supports this." The user takes explicit responsibility for their proxy's capabilities.

There's also a global kill switch: `DISABLE_EXPERIMENTAL_BETAS` forces standard mode regardless of other settings. When this is set, the system strips beta-specific fields from tool schemas before sending them to the API, ensuring no `defer_loading` or `tool_reference` reaches the wire. This was itself motivated by a separate failure: the kill switch originally didn't remove all beta headers, breaking LiteLLM-to-Bedrock proxies that rejected unknown beta flags.

### Pending MCP Servers

MCP servers connect asynchronously. When a user starts Claude Code, some servers may still be initializing. If tool search is enabled but no deferred tools exist yet (because no servers have connected), the system normally disables tool search for that request — there's nothing to search.

But if MCP servers are pending, it keeps ToolSearch available:

```
if useToolSearch AND no deferred tools AND no pending MCP servers:
    useToolSearch = false   # nothing to search, save a tool slot

if useToolSearch AND no deferred tools AND pending MCP servers:
    # keep ToolSearch — tools will appear when servers connect
```

When the model calls ToolSearch and no tools match, the result includes the names of pending servers:

```
{
    matches: [],
    total_deferred_tools: 0,
    pending_mcp_servers: ["github", "slack"]
}
```

This tells the model "your search found nothing, but these servers are still connecting — try again shortly."

### Cache Invalidation

Tool descriptions are memoized to avoid recomputing them on every search. But the deferred tool set can change mid-conversation (MCP server connects, tools added/removed). The cache key is the sorted, comma-joined list of deferred tool names. When the set changes, the cache clears:

```
function maybeInvalidateCache(deferredTools):
    currentKey = sorted(tool.name for tool in deferredTools).join(",")
    if currentKey != cachedKey:
        clearDescriptionCache()
        cachedKey = currentKey
```

The token count is also memoized with the same key scheme. This means connecting a new MCP server triggers one fresh token count and one fresh description computation, then subsequent searches reuse the cache.

### Tool Search Disabled Mid-Conversation

If the model switches from a supported model (Sonnet) to an unsupported one (Haiku) mid-conversation, the message history may contain `tool_reference` blocks that the new model can't process. The system handles this by stripping tool-search artifacts:

```
if not useToolSearch:
    for message in apiMessages:
        if message is user:
            stripToolReferenceBlocks(message)
        if message is assistant:
            stripCallerField(message)   # tool_use caller metadata
```

This ensures the API never receives `tool_reference` blocks when the current model doesn't support them, even if a previous model generated them.

There's an additional stripping path for a subtler failure: MCP server disconnection. If a server disconnects mid-conversation, previously valid `tool_reference` blocks now point to tools that don't exist in the current pool. The API rejects these with "Tool reference not found in available tools." The normalization pipeline strips `tool_reference` blocks for tools that aren't in the current available set, even when tool search is otherwise enabled.

### The Turn Boundary Problem

When the API server receives a `tool_result` containing `tool_reference` blocks, it expands them into a `<functions>` block — the same format used for tool definitions at the start of the prompt. This expansion happens server-side, and it creates an unexpected problem in the wire format.

The expanded `<functions>` block appears inline in the conversation. If the same user message that contains the `tool_result` also has text siblings (auto-memory reminders, skill instructions, etc.), those text blocks render as a second `Human:` turn segment immediately after the `</functions>` closing tag. This creates an anomalous pattern in the conversation structure: two consecutive human turns with a functions block in between.

The model learns this pattern. After seeing it several times in a conversation, it starts completing the pattern: when it encounters a bare tool result at the tail of the conversation (no text siblings), it emits the stop sequence instead of generating a meaningful response. The conversation just... stops. An A/B experiment with five arms confirmed the dose-response: more tool_reference messages with text siblings → higher stop-sequence rate.

Two mitigations work in concert:

**Turn boundary injection**: When a user message contains `tool_reference` blocks and no text siblings, the system injects a minimal text block (`"Tool loaded."`) as a sibling. This creates a clean `Human: Tool loaded.` turn boundary that prevents the model from seeing a bare functions block at the tail.

**Sibling relocation**: When a user message contains `tool_reference` blocks AND has text siblings (from auto-memory, attachments, etc.), the system moves those text blocks to the next user message that has `tool_result` content but NO `tool_reference`. This eliminates the anomalous two-human-turns pattern. If no valid target exists (the tool_reference message is near the end of the conversation), the siblings stay — that's safe because a tail ending in a human turn gets a proper assistant cue.

### Schema-Not-Sent Recovery

Sometimes the model tries to call a deferred tool without first discovering it via ToolSearch. This happens when the model hallucinates having seen the tool's schema (perhaps from its training data) or when a prior discovery was lost to compaction. The call fails at input validation — the model sends parameters that don't match any known schema, because the schema was never sent.

The raw validation error ("expected object, received string") doesn't tell the model what went wrong. So the system checks: is this a deferred tool that wasn't in the discovered set? If yes, it appends a hint:

```
"This tool's schema was not sent to the API — it was not in the
discovered-tool set. Use ToolSearch to load it first:
ToolSearch({ query: 'select:<tool_name>' })"
```

This turns a confusing Zod error into an actionable instruction. The model reads the hint, calls ToolSearch, gets the schema, and retries — one extra turn instead of a conversation-ending failure.

### Invisible by Design

ToolSearch calls never appear in the user's terminal output. The tool's `renderToolUseMessage` returns null and its `userFacingName` returns an empty string. In the message collapse system — which groups consecutive reads and searches into compact "Read 5 files" summaries — ToolSearch is classified as "absorbed silently": it joins a collapse group without incrementing any counter. The user sees "Read 3 files, searched 2 files" but the ToolSearch call that loaded the tool definitions is invisible.

This is deliberate. ToolSearch is infrastructure, not user-facing functionality. Showing "Searched for tools" in the output would be confusing — the user asked to create a GitHub issue, not to search for tools. The tool discovery is an implementation detail of how the model accesses MCP tools, and the UI hides it accordingly.

---

## The Complete Pipeline

Here's the full sequence for a single API request:

1. **Mode check**: Determine if tool search is `tst`, `tst-auto`, or `standard`.

2. **Model check**: Verify the model supports `tool_reference` blocks. If not, disable.

3. **Availability check**: Confirm ToolSearch is in the tool pool (not disallowed).

4. **Threshold check** (tst-auto only): Count deferred tool tokens via API (or character heuristic fallback). Compare to `floor(contextWindow × 10%)`.

5. **Build deferred set**: Mark each tool as deferred or not via the priority checklist.

6. **Scan history**: Extract discovered tool names from `tool_reference` blocks and compact boundary snapshots.

7. **Filter tools**: Include non-deferred tools, ToolSearch, and discovered deferred tools. Exclude undiscovered deferred tools.

8. **Serialize schemas**: Add `defer_loading: true` to deferred tools. Add beta header.

9. **Announce pool**: Prepend deferred tool list or compute delta attachment.

10. **Send request**: API receives full definitions with `defer_loading`, shows only non-deferred and discovered tools to the model.

11. **Model searches**: Calls ToolSearch with a query. Gets `tool_reference` blocks back.

12. **Next turn**: Step 6 finds the new references. Step 7 includes the discovered tools. The model can now call them.

13. **Compaction**: Before summarizing, snapshot discovered tools to boundary marker. After compaction, step 6 reads the snapshot.

Each step fails toward loading more tools, not fewer. Unknown model? Load everything. Token count unavailable? Use conservative heuristic. Proxy detected? Load everything unless explicitly opted in. The worst case is wasting tokens on tool definitions. The best case is saving 90% of tool definition tokens while maintaining full functionality through on-demand discovery.

The system turns an O(N) per-turn cost into O(1) for idle tools and O(k) for the k tools actually used in a conversation. For a user with 200 MCP tools who typically uses 5–10 per session, that's a 95% reduction in tool definition tokens — context space reclaimed for actual work.

---

## Design Trade-offs

Every engineering decision in this system reflects a trade-off. Here are the ones worth understanding:

**Deferral granularity**: Why defer by tool, not by MCP server? Server-level deferral would mean discovering one tool loads all tools from that server. This is simpler but wasteful — a GitHub server might have 40 tools, and you only need 3. Tool-level deferral uses more search turns but saves more tokens. The scoring system mitigates the extra turns: a single keyword search for "github" returns the most relevant tools, not all 40.

**Negative vs. positive model list**: The unsupported model list (`["haiku"]`) means every new model gets tool search by default. The alternative — a positive list of supported models — would mean every new model launch requires a code update. The negative list risks sending `tool_reference` blocks to a model that can't handle them, but the API would return a clear error, and the feature flag system can add models to the unsupported list within minutes.

**Token counting precision**: The character-per-token heuristic (2.5) is intentionally imprecise. Why not always use the API's token counter? Because the counter requires a network round-trip that might fail or add latency. The heuristic runs instantly. And the cost of over-counting (deferring when unnecessary) is one extra search turn. The cost of under-counting (not deferring when needed) is 60,000 wasted tokens per turn. The asymmetry favors the conservative heuristic.

**Cache key design**: Both the description cache and token count cache use the sorted tool name list as key, not a hash. This means cache comparison is O(N) in the number of deferred tools, but N is typically <200 and the comparison runs once per API request. A hash would be O(1) but risks collisions, and debugging cache issues with hashed keys is harder than with readable name lists.

**Snapshot vs. protection**: Why snapshot discovered tools instead of protecting `tool_reference` messages from compaction? The snip compaction strategy does protect these messages, but full compaction summarizes everything. Protecting individual messages from full compaction would fragment the summary and reduce its quality. The snapshot approach lets compaction work normally and reconstructs the discovery state from metadata.
