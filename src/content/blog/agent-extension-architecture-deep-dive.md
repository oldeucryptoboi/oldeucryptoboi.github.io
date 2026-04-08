---
title: "How Claude Code Extends Itself: Skills, Hooks, Agents, and MCP"
description: "Six extension systems with layered trust — CLAUDE.md, hooks, skills, the tool pool, MCP servers, and agents. How each solves a different problem safely."
pubDate: 2026-04-07
tags: ["Claude Code", "Anthropic", "Architecture", "MCP", "AI Agents"]
---

## The Problem

You want Claude Code to know your team's conventions, run your linter after every edit, delegate research to a background worker, and call your internal APIs through custom tools. These are four different extension problems, and the naive approach — one plugin system that does everything — fails because each problem has a fundamentally different trust profile.

Consider a team's coding conventions. These are passive instructions — text the model reads but never executes. They need no sandbox, no permissions, no isolation. Now consider a linter that runs after every file write. This is active code that executes on your machine in response to the model's actions. It needs a trust boundary: what if a malicious project's config file registers a hook that exfiltrates your SSH keys? Now consider a background research agent. It needs its own conversation, its own tool access, its own abort controller — but it must not silently approve dangerous operations. And a custom tool server? It's a separate process speaking a protocol, potentially remote, potentially untrusted.

One extension system can't handle all of these safely. Passive instructions with no execution risk get the same UX as remote tool servers that can exfiltrate data? That's either too permissive for tools or too restrictive for instructions.

The design principle is **layered trust with fail-closed defaults**. Each extension type gets exactly the trust boundary its threat model requires. Instructions are injected as text — no execution, no permissions needed. Hooks execute deterministic code — sandboxed, workspace-trust-gated, exit-code-based control flow. Agents get isolated conversations with scoped tool access — permission prompts bubble to the parent. Tool servers run out-of-process with namespaced capabilities and enterprise policy controls. Unknown extension types don't silently succeed — they don't exist.

This article traces six extension systems in execution order: CLAUDE.md (instructions), hooks (lifecycle callbacks), skills (reusable prompts), the tool pool (built-in + external), MCP (external tool servers), and agents (delegated execution). Each one exists because the others can't solve its problem safely.

---

## Layer 1: CLAUDE.md — Instructions as Text

### The Problem It Solves

Every project has conventions. "Use bun, not npm." "Always run tests before committing." "Never modify the migration files directly." These need to reach the model on every turn, survive context compaction, and compose across nested directories — without executing anything.

### How Discovery Works

Imagine you're working in `/home/alice/projects/myapp/src/components/`. The system walks upward:

```
/home/alice/projects/myapp/src/components/
/home/alice/projects/myapp/src/
/home/alice/projects/myapp/
/home/alice/projects/
/home/alice/
```

At each directory, it looks for three things:
- `CLAUDE.md` (checked-in project instructions)
- `.claude/CLAUDE.md` (same, nested in config dir)
- `.claude/rules/*.md` (individual rule files)

But not all directories are equal. The full discovery hierarchy has six tiers, loaded in order from lowest to highest priority:

```
1. Managed      — /etc/claude-code/CLAUDE.md (enterprise policy, always loaded)
2. User         — ~/.claude/CLAUDE.md (your personal global instructions)
3. Project      — CLAUDE.md files found walking up from cwd
4. Local        — CLAUDE.local.md (gitignored, private per-developer)
5. AutoMemory   — ~/.claude/projects/.../memory/MEMORY.md (persistent learning)
6. TeamMemory   — Shared team memory (experimental)
```

Priority matters because the model pays more attention to later content. Your project's "use bun" instruction at tier 3 takes precedence over a user-level "use npm" at tier 2. Enterprise policy at tier 1 is loaded first but can't be overridden by anything below it — it's structurally guaranteed to be present.

### The Include System

A CLAUDE.md can reference other files:

```markdown
# Project Rules
@./docs/coding-standards.md
@./docs/api-conventions.md
```

The `@` directive pulls in external files as separate instruction entries. Resolution rules:
- `@./relative` — relative to the including file's directory
- `@~/path` — relative to home
- `@/absolute` — absolute path

Circular includes are tracked by recording every processed path in a set. If file A includes B and B includes A, the second inclusion is silently skipped.

Security: only whitelisted text file extensions are loadable — over 100 extensions covering code, config, and documentation formats. Binary files (images, PDFs, executables) are rejected. This prevents a crafted include path from loading arbitrary binary data into the model's context.

### Conditional Rules

Rule files can have frontmatter that restricts when they activate:

```markdown
---
paths: src/api/**
---
Never use raw SQL queries in API handlers. Always use the query builder.
```

This rule only appears when the model is working on files matching `src/api/**`. The matching uses gitignore-style patterns — the same library that handles `.gitignore`, so glob semantics are consistent. Rules without a `paths` field apply unconditionally.

### How Instructions Reach the Model

All discovered files are concatenated into a single block, wrapped in a system-reminder tag, and injected as part of a user message — not the system prompt. This is a deliberate choice: system prompt content is cached aggressively, but CLAUDE.md content can change between turns (the user might edit a file). By injecting it as user-message content, it gets re-read on every turn without invalidating the system prompt cache.

The instruction block carries a header that tells the model these instructions override default behavior — a prompt-level enforcement that complements the structural priority ordering.

### Fail-Closed Properties

- Unknown file extensions in `@include` → silently skipped (no binary loading)
- File read errors (ENOENT, EACCES) → silently skipped (missing files don't crash)
- Circular includes → tracked and deduplicated
- Frontmatter parse errors → content loaded without conditional filtering (fail-open on conditions, fail-closed on content)
- HTML comments → stripped (authorial notes don't reach the model)
- AutoMemory → truncated after 200 lines (prevents unbounded context growth)

### Trade-Off: Safety Over Convenience

External includes (files outside the project root) require explicit approval. A CLAUDE.md in a cloned repository can't silently `@/etc/passwd` to exfiltrate system files into the model's context. The user must approve external includes once per project — a one-time friction that prevents a class of supply-chain attacks where a malicious repo's instructions load sensitive files.

---

## Layer 2: Hooks — Deterministic Lifecycle Callbacks

### The Problem It Solves

You want to run your linter after every file write. You want to block the model from committing to main. You want to send a webhook when a session ends. These are deterministic actions — no LLM judgment needed — that execute in response to specific lifecycle events.

### The Attack That Shaped the Design

Early in development, a vulnerability was discovered: a project's `.claude/settings.json` could register SessionEnd hooks that executed when the user declined the workspace trust dialog. The user says "I don't trust this workspace" and the workspace's code runs anyway. This led to a blanket rule: **all hooks require workspace trust**. In interactive mode, no hook executes until the user has explicitly accepted the trust dialog.

### Hook Events

Hooks fire at ~28 lifecycle points. The most important ones:

```
PreToolUse    — Before any tool executes (can block, modify input, or allow)
PostToolUse   — After successful tool execution (can inject context)
Stop          — Before the model stops (can force continuation)
SessionStart  — When a session begins
SessionEnd    — When a session ends (1.5-second timeout, not 10 minutes)
Notification  — When the system sends a notification
```

Each event carries structured JSON input — the tool name, the tool's input, session IDs, working directory, and more.

### Four Hook Types

**Command hooks** spawn a shell process (bash or PowerShell). The hook's JSON input is written to stdin. The process's exit code determines the outcome:

```
Exit 0  →  Success (continue normally)
Exit 2  →  Blocking error (prevent the action)
Exit 1  →  Non-blocking error (log and continue)
```

If the process writes JSON to stdout matching the hook output schema, that JSON controls behavior — permission decisions, additional context, modified tool input. If stdout isn't JSON, it's treated as plain text feedback.

A concrete example: a PreToolUse hook that blocks dangerous git operations:

```bash
#!/bin/bash
# Read JSON input from stdin
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name')
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if [ "$TOOL" = "Bash" ] && echo "$COMMAND" | grep -q "git push.*--force"; then
  echo '{"decision": "block", "reason": "Force push blocked by policy"}'
  exit 2
fi
exit 0
```

The exit code and JSON output are redundant by design — either mechanism can block. Exit code 2 without JSON still blocks. JSON `{"decision": "block"}` without exit code 2 still blocks. This redundancy means a hook that crashes mid-output (writing partial JSON) still has the exit code as a fallback signal.

On Windows, command hooks run through Git Bash, not cmd.exe. Every path in environment variables is converted from Windows format (`C:\Users\foo`) to POSIX format (`/c/Users/foo`) — Git Bash can't resolve Windows paths. PowerShell hooks skip this conversion and receive native paths.

**Prompt hooks** send the hook input to a fast model (Haiku by default) with a structured output schema: `{ok: boolean, reason?: string}`. No tool access. 30-second timeout. The LLM evaluates whether the action should proceed — useful when the decision requires judgment ("is this API call secure?") rather than deterministic checking. Thinking is disabled to reduce cost and latency.

**Agent hooks** are multi-turn: they spawn a restricted agent that can use tools (Read, Bash) to investigate, then must call a synthetic output tool with `{ok, reason}`. 60-second timeout, 50-turn limit. The agent can read test output, check file contents, then make a judgment. Its tool pool is filtered — no subagent spawning, no plan mode — to prevent recursive agent creation. If the agent hits 50 turns without producing structured output, it's cancelled silently — a fail-safe against infinite loops.

**HTTP hooks** POST the JSON input to a URL. SSRF protection blocks private/link-local IP ranges (except loopback). No redirects are followed (`maxRedirects: 0`). Header values support environment variable interpolation, but only from an explicit allowlist — `$SECRET_TOKEN` only resolves if `SECRET_TOKEN` is in the hook's `allowedEnvVars` array. Unresolved variables expand to empty strings, preventing accidental exfiltration. CRLF and NUL bytes are stripped from header values to prevent header injection attacks.

HTTP hooks are blocked for SessionStart and Setup events in headless mode — the sandbox callback would deadlock because the structured input consumer hasn't started yet when these hooks fire.

### Pattern Matching

Hooks can filter by event subtype. A PreToolUse hook with matcher `"Write|Edit"` only fires for file writes and edits. Matchers support:
- Simple strings: `"Write"` (exact match)
- Pipe-separated: `"Write|Edit"` (multiple exact matches)
- Regex patterns: `"^Bash.*"` (full regex)

An additional `if` condition supports permission-rule syntax: `"Bash(git *)"` only fires for bash commands starting with `git`.

### Aggregation and Priority

Multiple hooks can fire for the same event. Results are aggregated with a strict priority:

```
1. Any hook returns "deny"    → action is blocked (deny wins)
2. Any hook returns "allow"   → action is allowed (if no deny)
3. Any hook returns "ask"     → prompt the user
4. Default                    → normal permission flow
```

A single deny from any hook overrides all allows. This is the fail-closed property: a security hook can't be overridden by a convenience hook.

### Configuration Snapshot

Hook configurations are captured at startup into a frozen snapshot. Settings changes during the session update the snapshot, but the hooks that actually execute come from this snapshot — not from a live re-read of settings files. This prevents a TOCTOU attack where a process modifies `.claude/settings.json` between the trust check and hook execution.

Enterprise policy can lock hooks to managed-only (`allowManagedHooksOnly`), meaning only admin-defined hooks execute. Non-managed settings can't override this — the check happens in the snapshot capture, not at execution time.

### Trade-Off: Safety Over Convenience

SessionEnd hooks get a 1.5-second timeout (configurable via environment variable), not the 10-minute default. The reasoning: session teardown must be fast. A hook that takes 30 seconds to run would make "close the terminal" feel broken. This means complex cleanup (uploading logs, syncing state) must be designed to complete quickly or run asynchronously — a constraint that occasionally frustrates users but keeps the exit path responsive.

---

## Layer 3: Skills — Reusable Prompt Modules

### The Problem It Solves

You have a 500-line review checklist, a commit message template, or a complex deployment procedure. You want the model to follow it exactly when invoked, but you don't want it consuming context on every turn.

### Progressive Disclosure

Skills use a three-level disclosure strategy to manage context:

**Level 1 — Metadata only (always loaded):** The skill's name, description, and `when_to_use` field are injected into the system prompt's skill listing. This costs ~50-100 tokens per skill. A budget cap (1% of context window, ~8KB) limits total skill metadata — if you have 200 skills, descriptions get truncated. Bundled skills (compiled into the binary) are never truncated; user skills are truncated first.

**Level 2 — Tool prompt:** When the model decides to invoke a skill, it calls the Skill tool with the skill name. The tool validates the name, checks permissions, and returns a "launching skill" placeholder.

**Level 3 — Full content:** The skill's complete markdown body is loaded, argument substitution is applied (`$1`, `$2`, `${CLAUDE_SESSION_ID}`), inline shell commands are executed (if not from an MCP source), and the result is injected as new conversation messages. Only now does the full 500-line checklist enter the context.

This means 200 skills cost ~8KB of ongoing context, and only the invoked skill's full body enters the conversation.

### Skill Format

A skill lives in a directory: `.claude/skills/my-skill/SKILL.md`. The file uses YAML frontmatter:

```markdown
---
description: Review code for security vulnerabilities
allowed-tools: Bash, Read, Grep
model: opus
paths: src/security/**
context: fork
---

Review the following code for OWASP Top 10 vulnerabilities...
```

Key frontmatter fields:
- `allowed-tools` — which tools the skill can use (added to permission rules)
- `model` — model override (`opus`, `sonnet`, `haiku`, or `inherit`)
- `paths` — conditional activation (skill only available when working on matching files)
- `context: fork` — execute in an isolated subagent instead of inline
- `user-invocable` — whether the user can type `/skill-name` (default: true)
- `hooks` — scoped hooks that only apply during skill execution

### Conditional Skills

Skills with `paths` frontmatter start dormant. They're stored in a separate map, not exposed to the model. When a file operation touches a path matching the skill's pattern, the skill activates — it moves to the dynamic skills map and becomes available. This is the same gitignore-style matching used by CLAUDE.md conditional rules.

Why not just load all skills? Token budget. A project with 50 path-specific skills would waste context on skills irrelevant to the current work. Conditional activation means the model only sees skills relevant to the files it's actually touching.

### Dynamic Discovery

When the model reads or writes a file in a subdirectory, the system walks upward from that file looking for `.claude/skills/` directories. Newly discovered skill directories are loaded and merged into the dynamic skills map. This enables monorepo patterns where each package has its own skills.

Security: discovered directories are checked against `.gitignore`. A skill directory inside `node_modules/` is skipped — this prevents dependency packages from injecting skills.

### Inline Shell Execution

Skills can contain inline shell commands using `!` syntax:

```markdown
Current git branch: !`git branch --show-current`
```

When the skill body is loaded, these commands execute and their output replaces the command syntax. MCP-sourced skills (remote, potentially untrusted) have shell execution disabled entirely — a hard security boundary. The check is a simple conditional: if the skill's `loadedFrom` field is `'mcp'`, shell execution is skipped.

### Permission Model

The first time a skill is invoked by the model, the user is prompted. The permission check supports:
- Deny rules (exact or prefix match) → block permanently
- Allow rules (exact or prefix match) → allow permanently
- "Safe properties" auto-allow → skills that only set metadata (model, effort) and don't add tools or hooks are auto-approved

Default: ask. Unknown skills always prompt.

### Bundled Skill Security

Skills compiled into the binary extract their reference files to a temporary directory at runtime. The extraction uses `O_EXCL | O_NOFOLLOW` flags (POSIX) — the file must not already exist and symlinks are rejected. A per-process nonce in the directory path prevents pre-created symlink attacks. Path traversal protection rejects absolute paths and `..` components.

---

## Layer 4: The Tool Pool — Assembly and Permissions

### The Problem It Solves

The model needs a unified set of tools — built-in (Read, Write, Bash, Agent) plus external (MCP servers, IDE integrations). But which tools are available, and who controls access?

### Assembly

The tool pool is assembled from two sources:

```
built_in_tools = get_registered_tools(permission_context)
mcp_tools = filter_by_deny_rules(all_mcp_tools, permission_context)
pool = deduplicate(sort(built_in_tools) + sort(mcp_tools), by_name)
```

Three properties are maintained:
1. **Built-ins always win** — if an MCP tool has the same name as a built-in, the built-in takes precedence (deduplication preserves first occurrence)
2. **Stable sort order** — tools are sorted alphabetically within each partition, keeping built-ins as a contiguous prefix. This is critical for prompt caching: the server places a cache breakpoint after the last built-in tool. If MCP tools interleaved with built-ins, adding one MCP tool would invalidate all cached tool definitions downstream.
3. **Deny rules are absolute** — a tool in the deny list is removed regardless of source

### MCP Tool Namespacing

External tools are namespaced to prevent collisions:

```
mcp__github__create_issue
mcp__jira__create_ticket
```

The pattern is `mcp__<server>__<tool>`. Server and tool names are normalized: dots, spaces, and special characters become underscores. This namespacing means an MCP server can't shadow a built-in tool — `mcp__evil__Read` is a different tool from `Read`.

### IDE Tool Filtering

IDE extensions connect via MCP but have restricted access. Only two specific IDE tools are exposed to the model — the rest are blocked. This prevents an IDE extension from registering a tool named `Bash` that bypasses the bash security analyzer.

---

## Layer 5: MCP — External Tool Servers

### The Problem It Solves

You want to give the model access to your internal APIs, databases, or third-party services. These capabilities live in separate processes — potentially remote — and need their own lifecycle, authentication, and error recovery.

### Transport Types

MCP servers connect via six transport types:
- **stdio** — local child process (default, most common)
- **SSE** — Server-Sent Events (authenticated remote)
- **HTTP** — Streamable HTTP (MCP spec 2025-03-26)
- **WebSocket** — bidirectional streaming
- **SDK** — in-process (managed by the SDK)
- **claude.ai proxy** — remote servers bridged through a proxy with OAuth

### Configuration Hierarchy

Like CLAUDE.md, MCP server configs merge from multiple sources:

```
Enterprise    → exclusive control when present (blocks all others)
Local         → .claude/mcp.json in working directory
Project       → claude.json in project root
User          → ~/.claude/mcp.json
Dynamic       → SDK-provided servers
```

When an enterprise config exists, it has total control. Other scopes are blocked. This is the nuclear option for organizations that need to control exactly which external services the model can access.

### Enterprise Allowlist/Denylist

Policy settings define three types of allowlist entries:
- **Name-based**: `{serverName: "github"}`
- **Command-based**: `{serverCommand: ["node", "path/to/mcp.js"]}` (for stdio servers)
- **URL-based**: `{serverUrl: "https://mcp.example.com"}` (for remote servers)

The denylist always wins. A server matching any deny entry is blocked regardless of allowlist membership. If the allowlist exists but is empty, all servers are blocked. If the allowlist is undefined, all servers are allowed. This three-state logic (undefined/empty/populated) gives administrators precise control.

### Connection and Timeout

Servers are connected with a 30-second timeout. Connection is batched: 3 local servers in parallel, 20 remote servers in parallel. If a server fails to connect, it enters a failure state but doesn't block other servers.

Tool calls have a separate timeout — nearly 28 hours by default (configurable). This allows long-running operations (database migrations, large builds) without arbitrary cutoffs. Progress is logged every 30 seconds so the user knows something is happening.

### Session Expiry and Recovery

Remote servers have stateful sessions. When a session expires, the server returns a 404 with JSON-RPC error code -32001, or the connection closes with error -32000. The client detects both cases, clears the connection cache, and throws a session-expired error. The next tool call will transparently reconnect.

Authentication failures (401) follow a parallel path: the client status updates to "needs-auth," tokens are cached with a 15-minute TTL, and the next connection attempt triggers a token refresh. OAuth flows support step-up authentication — a 403 response triggers a re-authentication challenge before the SDK's default handler fires.

A more subtle failure: URL elicitation. Some MCP servers require the user to visit a URL to authorize an action (OAuth consent, MFA challenge). The server returns error code -32042 with a completion URL. The client emits an elicitation request, waits indefinitely for the user to complete the flow, then retries the original tool call. This is a blocking wait — but since it's triggered by a user-facing auth requirement, the blocking is intentional.

### Error Boundaries

MCP server errors never contain sensitive data. All error messages are wrapped in a telemetry-safe type that strips user code and file paths. Server stderr is buffered to a 64 MB cap to prevent unbounded memory growth from a chatty or malicious server. When a stdio server crashes (ECONNRESET), the error message says "Server may have crashed or restarted" — not the actual stderr contents.

---

## Layer 6: Agents — Delegated Execution

### The Problem It Solves

You want the model to research a codebase in the background while you keep working. You want it to delegate a complex task to a specialist (an "Explore" agent that only searches, a "Plan" agent that only designs). You want multiple agents working in parallel on different parts of a refactor.

### Three Execution Models

**Synchronous subagents** share the parent's abort controller. When the user presses Ctrl+C, both parent and child stop. The child's state mutations (tool approvals, file reads) propagate to the parent via shared `setAppState`. The child runs inline — the parent waits for it to finish.

**Async background agents** get their own abort controller. The parent continues working. The child's state mutations are isolated — a separate denial counter, separate tool decisions. When the child finishes, its result is delivered as a notification. Permission prompts are auto-denied (the child can't show UI) unless the agent runs in "bubble" mode, where prompts surface in the parent's terminal.

**Teammates** are full separate processes (via tmux split-pane or iTerm2) or in-process runners isolated via AsyncLocalStorage. Each teammate has its own conversation history, its own model, its own abort controller. Communication happens through a file-based mailbox — JSON messages written to a shared team directory. The team lead writes a prompt to a teammate's inbox; the teammate polls it.

### Context Isolation

Every agent gets its own `ToolUseContext` — a structure containing the conversation, tool pool, permissions, abort controller, file state cache, and callbacks. The isolation strategy:

```
readFileState     → cloned (cache sharing for prompt cache hits)
abortController   → shared (sync) or new (async)
setAppState       → shared (sync) or no-op (async)
messages          → stripped for teammates (they build their own)
tool decisions    → fresh (no leaking parent's approve/deny history)
MCP clients       → merged (parent + agent-specific servers)
```

The critical insight is that cloning `readFileState` isn't about correctness — it's about cache hits. When a forked agent makes an API call, the server checks whether the message prefix matches a cached prefix. If the fork and parent have different file state caches, they'll make different tool-result replacement decisions, producing different message bytes and missing the cache. Cloning ensures byte-identical prefixes.

### Cache-Safe Forking

After every turn, the parent saves its "cache-safe parameters" — system prompt, user context, system context, tool definitions, and conversation messages. When a fork is created, it retrieves these parameters and uses them directly. The fork's API request starts with a byte-identical prefix, and only the fork's new prompt differs. The server recognizes the shared prefix and reads it from cache — potentially saving 90%+ on input costs for the fork.

This is why fork children inherit the parent's exact tool pool (`useExactTools: true`) and thinking config. Changing even one tool definition would alter the tool schema bytes, breaking the prefix match.

### Tool Filtering

Each agent definition can specify allowed and disallowed tools:

```
tools: [Read, Grep, Glob, Bash]          → only these tools available
disallowed_tools: [Write, Edit, Agent]    → these removed from pool
```

The resolution:
1. Start with the full tool pool
2. If `tools` is specified and not `['*']`, filter to only listed tools (plus always-included tools like the stop tool)
3. Remove any tools in `disallowed_tools`
4. Remove agent-disallowed tools (Agent tool itself for non-fork agents, plan mode tools)

Read-only agents like Explore and Plan additionally skip CLAUDE.md (saves ~5-15 Gtok/week fleet-wide) and git status (stale snapshot, they'll run `git status` themselves if needed).

### Permission Bubbling

When an agent needs a permission decision:
- **Sync agents**: The prompt surfaces in the parent's terminal. The user approves or denies. The decision propagates to the child's permission context.
- **Async agents in bubble mode**: Same as sync — the prompt surfaces in the parent's terminal, but the agent waits asynchronously. Automated checks (permission classifier, hooks) run first; the user is only interrupted when automation can't resolve it.
- **Async agents without bubble**: Permissions are auto-denied. The agent must work within its pre-approved tool rules.
- **Teammates**: Permission mode is inherited via CLI flags when spawning the process. `--dangerously-skip-permissions` propagates — but not when plan mode is required (a safety interlock).

### Fork Recursion Guard

Fork children keep the Agent tool in their tool pool (for cache-identical tool definitions), but recursive forking is blocked at call time. The system scans the conversation history for a boilerplate tag injected into every fork child's first message. If found, the agent is already a fork — further forking is rejected.

The boilerplate itself is instructive. Every fork child receives a message that begins:

```
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT — that's for
   the parent. You ARE the fork. Do NOT spawn sub-agents; execute directly.
2. Do NOT converse, ask questions, or suggest next steps
3. USE your tools directly: Bash, Read, Write, etc.
...
```

This prompt engineering is a defense-in-depth against the model's tendency to delegate. The system prompt (inherited from the parent for cache reasons) may contain instructions to fork work. The boilerplate overrides those instructions at the conversation level — later in the message sequence, higher priority.

### Worktree Isolation

Agents can be spawned with `isolation: "worktree"`, which creates a separate git worktree — a full copy of the repository on a separate branch. The agent operates in this isolated copy: writes don't affect the parent's files, and the parent's subsequent edits don't corrupt the agent's state.

When a worktree agent inherits conversation context from the parent, all file paths in that context refer to the parent's working directory. The system injects a notice telling the agent to translate paths, re-read files before editing (they may have changed since the parent saw them), and understand that changes are isolated.

### Max Turns and Cleanup

Every agent has a turn limit (default varies by agent type, capped by definition). When the limit is reached, the agent receives a `max_turns_reached` attachment and stops. The cleanup sequence:

```
1. Close agent-specific MCP servers (only newly created ones, not shared)
2. Remove scoped hooks registered by the agent's frontmatter
3. Clear prompt cache tracking state
4. Release cloned file state cache
5. Free conversation messages (GC)
6. Remove Perfetto trace registration
7. Clear transcript routing
8. Kill background bash tasks spawned by this agent
```

This cleanup happens in a `finally` block — it runs whether the agent succeeded, failed, or was aborted.

---

## The Full Pipeline

When you type a message, here's what happens to the extension systems:

```
1. CLAUDE.md files discovered and loaded (6-tier hierarchy)
   → Instructions injected as system-reminder in user message

2. UserPromptSubmit hooks fire
   → Can block the prompt, inject additional context, or modify it

3. System prompt assembled with skill metadata
   → ~50-100 tokens per skill, budget-capped at 1% of context

4. Tool pool assembled (built-in + MCP, sorted, deduplicated)
   → Deny rules applied, built-ins win on name conflict

5. Model generates response, calls tools
   → PreToolUse hooks fire before each tool (can block, allow, modify input)
   → PostToolUse hooks fire after each tool (can inject context)

6. Model invokes a Skill
   → Permission check → full body loaded → argument substitution
   → Shell commands executed (unless MCP source) → content injected

7. Model spawns an Agent
   → Isolated context created → tools filtered → MCP servers merged
   → Hooks scoped → query loop runs → results returned

8. Session ends
   → SessionEnd hooks fire (1.5-second timeout)
   → MCP servers disconnected → agent cleanup
```

Every layer is fail-closed. Unknown CLAUDE.md extensions are skipped. Unknown hook events are ignored. Unknown skill types are rejected. Unknown MCP tools are filtered by deny rules. Unknown agent types are blocked at validation. The system doesn't need to anticipate every new extension type — it only needs to correctly handle the ones it explicitly supports. Everything else gets a "no."

The alternative — a blocklist approach where you enumerate what's dangerous — means every new extension type is a zero-day. The allowlist approach means every new extension type starts with "ask the user." That's the fundamental trade-off: a slight friction on adoption in exchange for a structural guarantee that surprises are visible.

---

*Follow me on [X](https://x.com/oldeucryptoboi) — I post as @oldeucryptoboi.*
