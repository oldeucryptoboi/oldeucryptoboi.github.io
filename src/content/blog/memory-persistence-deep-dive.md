---
title: "How Claude Code Remembers (And Forgets): The Memory and Persistence Architecture"
description: "Five persistence layers from CLAUDE.md to session transcripts. How knowledge survives across sessions, what compaction destroys, and the wiki gap."
pubDate: 2026-04-09
tags: ["Claude Code", "Anthropic", "Architecture", "Memory", "AI Agents"]
---

# How Claude Code Remembers (And Forgets): The Memory and Persistence Architecture

Claude Code processes thousands of lines of code, generates insights, solves bugs, discovers architecture — then the session ends and it forgets everything. The next session starts from scratch. The model re-reads the same files, re-traces the same execution paths, re-discovers the same patterns. Nothing compounds.

This is the fundamental limitation of a context-window-only architecture. The context window is working memory: capacious, fast, but volatile. When it fills up, old content is compressed or discarded. When the session ends, everything goes.

The naive solution: just save everything to disk. But "everything" is too much. A 200-turn debugging session produces megabytes of tool calls, error messages, failed attempts, and dead ends. Loading all of that into the next session would waste most of the context window on irrelevant history. You need selectivity — keep the lessons, discard the scaffolding.

The opposite extreme: save nothing. Let the model re-derive knowledge from the codebase every session. This works for small projects but collapses at scale. A developer who's been working on a codebase for months has context that can't be re-derived from the code alone: why this architecture was chosen, what patterns the team prefers, which approaches were tried and abandoned, what the user's communication style is.

Claude Code takes a middle path. It has five persistence mechanisms, each operating at a different timescale and abstraction level: CLAUDE.md instruction files, an auto-memory directory with a typed file system, a background memory extraction agent, context compaction that summarizes old messages, and raw session transcripts. Together they form a layered persistence architecture — not a wiki, not RAG, but something in between that trades comprehensiveness for simplicity.

This article traces each layer: how it stores knowledge, what it discards, where it truncates, and what falls through the gaps.

---

## Layer 1: CLAUDE.md — The Instruction Layer

Before the model sees any user message, it loads a stack of instruction files. These are human-written (or human-edited) markdown files that tell the model how to behave in a specific project. They're the most persistent layer — they survive not just across sessions but across users.

### Discovery

The system discovers CLAUDE.md files by walking the filesystem in a specific order:

```
1. Managed: /etc/claude-code/CLAUDE.md
   (global admin instructions, all users)

2. User: ~/.claude/CLAUDE.md
   (private global instructions, all projects)

3. Project: walk from CWD up to root, in each directory check:
   - CLAUDE.md
   - .claude/CLAUDE.md
   - .claude/rules/*.md
   (committed to the codebase, shared with team)

4. Local: CLAUDE.local.md in each project root
   (gitignored, private to this developer)
```

Files are loaded in this order but **priority increases from bottom to top** — local files override project files, which override user files, which override managed files. The model sees them in reverse priority order and pays more attention to the last-loaded content.

### The @include Directive

CLAUDE.md files can reference other files using @ notation:

```
@./docs/coding-standards.md
@~/personal-preferences.md
@/absolute/path/to/instructions.md
```

Included files are added as separate entries before the including file. The system prevents circular references by tracking processed paths. Only text-format files are allowed — binary files (images, PDFs) are silently ignored.

### Trust Boundaries

Project-level CLAUDE.md files (`.claude/settings.json`) have restricted power compared to user-level files. A malicious repository could commit a CLAUDE.md that attempts to:

- Redirect the memory directory to `~/.ssh` to gain write access to sensitive files
- Set dangerous environment variables
- Override security-critical settings

The system prevents this by restricting which settings project files can modify. The auto-memory directory path, for instance, can only be set from user-level, local-level, or policy-level settings — never from project settings committed to a shared repo.

### The 40,000-Character Cap

Each CLAUDE.md file is capped at 40,000 characters. Beyond this, content is truncated. This prevents a project with an enormous instruction file from consuming the entire context window before the conversation even starts.

---

## Layer 2: Auto-Memory — The Persistent Knowledge Store

The auto-memory system is Claude Code's persistent knowledge base. It lives at `~/.claude/projects/<sanitized-project-root>/memory/` and contains markdown files that persist across sessions.

### The MEMORY.md Entrypoint

Every memory directory has a `MEMORY.md` file that serves as an index. It's loaded into the system prompt at the start of every session. The model sees it, and the model writes to it.

Two hard caps prevent MEMORY.md from consuming too much context:

```
MAX_LINES = 200
MAX_BYTES = 25,000  # ~125 chars/line
```

If either cap is exceeded, the content is truncated and a warning is appended:

```
> WARNING: MEMORY.md is 347 lines (limit: 200).
> Only part of it was loaded. Keep index entries to
> one line under ~200 chars; move detail into topic files.
```

The byte cap was added to catch a specific failure mode: "long-line indexes that slip past the line cap." Production telemetry showed the p100 (worst case) was a MEMORY.md at 197KB while staying under 200 lines — each line averaging ~1,000 characters. The line check passed. The context window ate 197KB of memory index. The 25KB byte cap catches this.

### The Truncation Algorithm

The truncation is a two-step process, and the order matters:

```
function truncateEntrypointContent(raw):
    lines = raw.trim().split('\n')
    lineCount = lines.length
    byteCount = raw.trim().length

    # Step 1: Truncate by lines (natural boundary)
    if lineCount > MAX_LINES:
        truncated = lines[0:MAX_LINES].join('\n')
    else:
        truncated = raw.trim()

    # Step 2: Truncate by bytes (catches long-line abuse)
    # BUT: cut at the last newline before the cap
    # so we don't slice mid-line
    if truncated.length > MAX_BYTES:
        cutPoint = truncated.lastIndexOf('\n', MAX_BYTES)
        truncated = truncated[0:cutPoint or MAX_BYTES]

    # Append warning naming WHICH cap fired
    # (line only, byte only, or both)
    return truncated + warning
```

A subtle design choice: the warning message names the *original* byte count, not the post-line-truncation byte count. This means the warning says "your file is 197KB" even though line truncation already reduced it. The user sees the real problem (lines are too long) rather than a misleading post-truncation size.

The byte truncation cuts at `lastIndexOf('\n', MAX_BYTES)` — it finds the last newline before the byte cap and cuts there, rather than slicing mid-line. If no newline exists before the cap (one enormous line), it falls back to a hard cut at the byte boundary.

### The mkdir Problem

An early failure mode: the model would burn turns running `ls` and `mkdir -p` before writing its first memory file. It didn't trust that the directory existed. The system now explicitly tells the model in the prompt:

```
This directory already exists — write to it directly
with the Write tool (do not run mkdir or check for
its existence).
```

The harness guarantees this by calling `ensureMemoryDirExists()` during prompt building. The mkdir is recursive and swallows `EEXIST`. If it fails for a real reason (permissions, read-only filesystem), the error is logged at debug level and the model's Write call will surface the actual error.

### The Index, Not a Memory

A critical design choice: MEMORY.md is an **index**, not a memory store. Each entry should be one line under ~150 characters — a title and a link to a topic file:

```
- [Testing preferences](testing.md) — always use vitest, prefer unit tests
- [Git workflow](git-workflow.md) — conventional commits, squash merges
```

The actual knowledge lives in separate topic files (`testing.md`, `git-workflow.md`). These are read on demand when relevant, not loaded into every session's context. This two-tier design keeps the always-loaded context small while allowing arbitrarily detailed knowledge in topic files.

### Typed Memory System

The system defines a taxonomy of memory types with structured frontmatter:

```
---
name: User testing preferences
type: preference
description: How the user wants tests written and run
---
```

The taxonomy has four types — not generic categories, but carefully scoped roles:

- **user**: Who the user is — role, expertise, goals. "Senior Go engineer, new to React" changes how the model explains frontend code. Always private (never shared with team memory).
- **feedback**: What the user corrected or confirmed. "Don't mock the database — we got burned when mocks passed but prod migration failed." Includes *why* so the model can judge edge cases, not just follow the rule blindly. The prompt explicitly instructs: record from success AND failure, not just corrections.
- **project**: Ongoing work, deadlines, decisions. "Merge freeze starts Thursday for mobile release." Must convert relative dates to absolute ("Thursday" → "2026-03-05") so the memory stays interpretable after time passes. These decay fast.
- **reference**: Pointers to external systems. "Pipeline bugs tracked in Linear project INGEST." These are bookmarks, not content.

Each type has structured guidance for when to save, how to use, and body structure (lead with fact, then "Why:" line, then "How to apply:" line). The prompt includes worked examples showing the model's expected behavior for each type.

### What NOT to Save

The instructions explicitly prohibit saving information that's derivable from the current project state:

- Code patterns visible in the codebase
- Architecture discoverable from the file structure
- Git history that git commands can retrieve
- Session-specific context (current task, in-progress work)
- Speculative or unverified conclusions

This constraint fights a specific failure mode: memory files that duplicate what the model can already see. A memory entry saying "the project uses React with TypeScript" is worse than useless — it wastes context on information the model can derive from `package.json` in seconds.

### Path Resolution

The auto-memory directory path is resolved through a three-step chain:

```
1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE env var
   (full-path override, used by Cowork/SDK)

2. autoMemoryDirectory in settings.json
   (trusted sources only: policy, local, user — NOT project)

3. ~/.claude/projects/<sanitized-git-root>/memory/
   (computed from canonical git root)
```

The first match wins. Step 1 exists for multi-agent orchestration (Cowork) where the per-session working directory contains the VM process name — every session would produce a different project key without the override. Step 2 lets users customize the path in their personal settings. Step 3 is the default.

The result is memoized, keyed on the project root. This prevents repeated filesystem operations: render-path callers fire per tool-use message per React re-render, and each miss would cost four `parseSettingsFile` calls (one per settings source), each involving `realpathSync` and `readFileSync`.

### Path Security

The memory directory path undergoes strict validation:

```
function validateMemoryPath(raw):
    reject if relative (starts with "../")
    reject if root or near-root (length < 3)
    reject if Windows drive root ("C:\")
    reject if UNC path ("\\server\share")
    reject if contains null byte
    reject if tilde expansion would resolve to $HOME
    normalize and add trailing separator
```

This prevents a settings file from redirecting the memory directory to sensitive locations. A particularly subtle attack: setting `autoMemoryDirectory: "~/"` would make `isAutoMemPath()` match everything under the home directory, granting the model write access to `~/.ssh`, `~/.gitconfig`, and other sensitive files. The validator rejects bare tilde expansions that would resolve to the home directory itself.

### Worktree Sharing

The memory directory key is derived from the **canonical git root**, not the current working directory. This means all git worktrees of the same repository share one memory directory. If you're working in `feature-branch` worktree and save a memory about testing preferences, the `main` worktree sees it too.

---

## Layer 3: Memory Extraction — The Background Agent

Manually saving memories requires the model to decide, mid-task, to stop and write knowledge to disk. This interrupts the task, consumes context tokens on memory management, and relies on the model prioritizing long-term knowledge over short-term task completion.

The memory extraction agent solves this by running **after** the main task completes. It's a forked agent — a perfect fork of the main conversation that shares the parent's prompt cache — triggered at the end of each query loop when the model produces a final response with no tool calls.

### How It Works

```
function executeExtractMemories(hookContext):
    # Skip if extract mode not active
    if not isExtractModeActive():
        return

    # Skip if the main agent already wrote memories this turn
    if hasMemoryWritesSince(lastMemoryMessageUuid):
        return

    # Skip if not enough context has accumulated
    if turnsSinceLastExtraction < threshold:
        return

    # Scan existing memory files for manifest
    manifest = scanMemoryFiles(autoMemDir)

    # Build extraction prompt with conversation context
    prompt = buildExtractPrompt(manifest, pendingContext)

    # Fork the agent with restricted tool access
    runForkedAgent({
        prompt: prompt,
        canUseTool: createAutoMemCanUseTool(),
        # ... shares parent's prompt cache
    })
```

### Tool Restrictions

The extraction agent is severely restricted:

- **Read tools**: Glob, Grep, Read — can search and read any file
- **Bash**: Read-only mode (no writes, no side effects)
- **Write/Edit**: Only to files within the auto-memory directory

This prevents a memory extraction bug from corrupting the project's source code. The agent can read anything to understand context, but can only write to memory files.

### Deduplication

The main agent has full save instructions in its prompt — it can write memories at any time. The extraction agent is the backup for when it doesn't. These two must be mutually exclusive per turn.

Detection works by scanning assistant messages after the last extraction cursor for Write or Edit tool calls targeting an auto-memory path. The check is simple: iterate messages after the cursor UUID, find assistant messages with `tool_use` blocks, extract the file path from the tool input, and test it against `isAutoMemPath()`.

If any memory write is found, the extraction agent skips entirely and advances its cursor past the range. The main agent's explicit save is trusted. If no memory write is found, the extraction agent forks and scans for anything the main agent missed.

A subtle edge case: if the cursor UUID was removed by context compaction (the message it pointed to was summarized away), the system falls back to counting all model-visible messages rather than returning zero. Returning zero would permanently disable extraction for the rest of the session — a silent failure mode that was caught and fixed.

### Feature Gates

Memory extraction is behind multiple feature gates: a compile-time `EXTRACT_MEMORIES` flag, a GrowthBook `tengu_passport_quail` runtime gate, and a throttling gate (`tengu_bramble_lintel`) that controls how often extraction runs. In non-interactive sessions (SDK, CI), extraction is disabled by default unless explicitly opted in.

### Memory vs. Plans vs. Tasks

The system prompt explicitly tells the model when NOT to use memory:

- **Plans** are for non-trivial implementation tasks where alignment with the user is needed. If you're about to start building something and want to confirm the approach, use a plan — don't save it to memory. Plans are session-scoped.
- **Tasks** are for breaking work into discrete steps and tracking progress within the current conversation. Tasks persist within the session but not across sessions.
- **Memory** is reserved for information useful in future conversations: user preferences, project conventions, lessons learned.

This separation fights a failure mode where the model saves everything to memory — including task lists, implementation plans, and debugging notes that are only relevant right now. Memory becomes a dump, not a knowledge base.

---

## Layer 4: Context Compaction — The Lossy Summarizer

When the context window fills up, Claude Code doesn't crash or stop. It compresses older messages into summaries, freeing space for new content. This is context compaction — and it's the most impactful persistence mechanism because it operates during every long session.

### Microcompact: The First Line of Defense

Before full compaction fires, the system tries a cheaper operation: clearing old tool results. Not all tool results — only results from specific tools that produce large, already-processed outputs:

```
COMPACTABLE_TOOLS = {
    FileRead, Bash, Grep, Glob, WebSearch,
    WebFetch, FileEdit, FileWrite
}
```

For each assistant message, the system collects tool-use IDs matching these tools, then replaces their corresponding tool-result content with `[Old tool result content cleared]`. This recovers tokens without losing semantic information — the model already processed these results and incorporated them into its reasoning.

Microcompact runs on a time-based schedule, not just at threshold. The system estimates token counts per message using a conservative 4/3 padding multiplier (since the estimation is approximate). Images and documents are estimated at a flat 2,000 tokens regardless of actual size.

### The Auto-Compact Threshold

The auto-compact trigger is not "~80% of the context window." It's more precise than that:

```
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000  # p99.99 of summary output
AUTOCOMPACT_BUFFER_TOKENS = 13,000

effective_window = context_window - MAX_OUTPUT_TOKENS_FOR_SUMMARY
threshold = effective_window - AUTOCOMPACT_BUFFER_TOKENS
```

For a 200K-token context window: effective = 180K, threshold = 167K. That's ~83% of the raw window, but the calculation is based on reserving output space, not a simple percentage.

The system also supports an environment variable (`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`) that sets the threshold as a percentage — useful for testing compaction behavior without filling the entire context window.

### The Full Compaction Pipeline

When the threshold is hit:

```
1. Pre-compact hooks: Execute user-defined pre-compact hooks

2. Fork a summary agent: Uses runForkedAgent (same pattern as
   memory extraction) to read old messages and produce a summary.
   Max output: 20,000 tokens.

3. Replace old messages: The summary becomes a "boundary message"
   — a system message that says "here's what happened before
   this point."

4. Post-compact cleanup: Strip images, clear stale attachments,
   prune tool reference blocks

5. Post-compact hooks: Execute user-defined post-compact hooks
```

### Recursion Guards

Compaction itself uses a forked agent that consumes context. If the compaction agent's own context fills up and triggers auto-compact inside the compaction fork, the system would deadlock. Three query sources are excluded from auto-compact: `session_memory`, `compact`, and the context-collapse agent (`marble_origami`). Each one would create a recursive loop if it triggered compaction.

### What Compaction Preserves

The boundary message includes metadata that downstream systems need:

- **User context**: CLAUDE.md content, memory files, git status (snapshotted at compaction time so it can be re-injected if the summary doesn't mention it)
- **Discovered tools**: Tools that were loaded via tool search before compaction (so they remain available after)
- **Message count**: How many messages were summarized (for analytics)
- **Trigger type**: Whether compaction was manual (`/compact`) or automatic

### What Compaction Loses

This is the critical limitation. Compaction is a **lossy** operation. The summary agent compresses dozens of messages into a paragraph. Details that seemed unimportant at compaction time are discarded:

- Specific error messages from failed attempts
- Exact file contents that were read
- The sequence of approaches tried and abandoned
- Tool call arguments and raw outputs
- Nuances in the user's instructions

A five-turn debugging session where the model read three files, tried two fixes, and discovered a subtle race condition gets summarized as: "Investigated race condition in worker pool. Fixed by adding mutex around shared counter." The specific files, the failed fix, the diagnostic reasoning — gone.

This is the opposite of the wiki pattern. A wiki would compile those details into a persistent artifact: a page for the race condition, cross-referenced with the worker pool architecture page, noting which approach failed and why. Compaction discards all of that to save tokens.

### The Circuit Breaker

Compaction can fail. The summary agent might produce an incomplete response, the API might return an error, or the summarized content might still exceed the context window. The system tracks consecutive failures:

```
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

if compaction fails 3 times in a row:
    stop attempting compaction entirely
```

This cap was added after telemetry revealed the cost of unbounded retries: 1,279 sessions had 50 or more consecutive compaction failures in a single session, with the worst reaching 3,272 consecutive failures. Globally, this wasted approximately 250,000 API calls per day — sessions stuck in a compact → fail → compact → fail loop, each attempt consuming tokens for the summary agent but never producing a usable result.

The failure modes that cause this are typically irrecoverable: `prompt_too_long` errors where even the compacted content exceeds the window, or API errors that persist regardless of retries. Three consecutive failures is enough to distinguish "transient error" from "structurally impossible."

A separate guard prevents a specific infinite loop: compact → still too long → error → stop hook blocking → compact → repeat. A boolean flag (`hasAttemptedReactiveCompact`) ensures reactive compaction fires at most once per error cycle.

---

## Layer 5: Session Transcripts — The Raw Archive

Every message in a Claude Code session is written to a JSONL file on disk. These are the raw, immutable transcripts — the equivalent of the "raw sources" layer in the wiki pattern.

### Where They Live

```
~/.claude/projects/<sanitized-project-root>/<session-uuid>.jsonl
```

Each line is a JSON object representing a message: user messages, assistant messages, tool calls, tool results, system messages, compaction boundaries. The complete session, including compressed content, is preserved.

### Searching Past Context

The memory system includes instructions for searching transcripts as a last resort:

```
## Searching past context

When looking for past context:
1. Search topic files in your memory directory:
   Grep with pattern="<search term>" path="<memory-dir>" glob="*.md"

2. Session transcript logs (last resort — large files, slow):
   Grep with pattern="<search term>" path="<project-dir>/" glob="*.jsonl"

Use narrow search terms (error messages, file paths, function names)
rather than broad keywords.
```

This is the only mechanism for accessing knowledge from previous sessions that wasn't explicitly saved to memory. It's a raw text search over potentially megabytes of JSON — not indexed, not structured, not semantic. The instructions explicitly call it a "last resort" and warn that it's slow.

### The Cost of Raw Storage

Session transcripts are the most complete persistence layer and the least useful. They contain everything — every tool call argument, every file content read, every failed attempt, every compaction boundary. A single long session can produce megabytes of JSONL.

But the only access mechanism is raw text search: grep for a pattern across all `.jsonl` files in the project directory. No indexing, no semantic search, no filtering by message type or tool name. The instructions explicitly call this a "last resort" and warn about speed. In practice, searching transcripts is useful for recovering specific error messages or file paths from previous sessions, but useless for answering questions like "what architectural decisions did I make last month?"

This is the raw-sources layer in the wiki pattern — comprehensive, immutable, and effectively inaccessible without a synthesis layer on top. The wiki pattern would build entity pages from these transcripts automatically. Claude Code leaves them as JSON on disk.

### Session Continuity

When a session is resumed (via `claude --continue`), the system loads the transcript from disk and replays it into the context window. If the transcript is longer than the context window, it triggers compaction to fit. This means long sessions that are resumed lose detail from their early turns — the compaction at resume time is an additional lossy step.

A resumed session re-appends session metadata (the original system prompt context, memory content, etc.) to ensure the model has the same starting context it would in a fresh session. But the compaction summary may omit details that the model relied on in earlier turns — a resumed session is always a degraded version of the original.

---

## Layer 6: The Assistant Daily Log (KAIROS Mode)

A separate persistence mode exists for long-lived assistant sessions. When KAIROS mode is active, the memory system switches from the index-and-topic-files model to an **append-only daily log**:

```
~/.claude/projects/<root>/memory/logs/2026/04/2026-04-09.md
```

The agent appends timestamped bullets to today's log file as it works. A separate nightly `/dream` skill distills these logs into topic files and updates MEMORY.md. This acknowledges that long-lived sessions produce too much context for real-time synthesis — the distillation happens offline.

The prompt for this mode is carefully designed for cache stability: it describes the log path as a **pattern** (`YYYY/MM/YYYY-MM-DD.md`) rather than today's literal date, because the system prompt is cached and not invalidated on date change. The model derives the current date from a separate attachment.

---

## What's Missing: The Wiki Gap

Andrej Karpathy's [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) proposes a three-layer architecture for LLM-maintained knowledge: raw sources (the codebase, documents, conversation logs), a wiki layer (persistent, interlinked entity pages maintained by the LLM itself), and a schema layer (instructions that teach the LLM how to maintain the wiki). Claude Code has the raw sources (the codebase on disk, session transcripts) and the schema (CLAUDE.md, memory type taxonomy). What it's missing is the wiki — a persistent, compounding knowledge artifact where every interaction makes the knowledge base richer.

Comparing Claude Code's persistence architecture to this pattern reveals specific gaps — not as criticism, but as a map of where knowledge fails to compound.

### No Cross-Referencing

Memory files are isolated. A file about "testing preferences" doesn't link to a file about "CI pipeline" even though they're related. There's no link graph, no backlinks, no mechanism for the model to discover connections between memories without reading every file.

### No Contradiction Detection

If session 1 saves "use vitest for testing" and session 50 saves "the project migrated to jest," both memories coexist. No system detects the contradiction. The model might follow either one depending on which it reads first.

### No Query-Time Filing

When the model answers a complex question — synthesizing information from five files, discovering an architectural insight, tracing a subtle bug — the answer dies with the session. There's no mechanism to say "this answer was valuable, file it as a wiki page." The next session will have to re-derive the same insight from scratch.

### No Lint or Health Check

There's no periodic audit of memory quality. No detection of stale entries, orphan files, missing frontmatter, or entries that contradict the current codebase. A memory file from six months ago saying "the API uses REST" might be wrong if the project migrated to gRPC, but nothing flags this.

### No Structured Index

MEMORY.md is a flat list. It has no categories, no hierarchy, no metadata beyond what the model chose to write. Compare this to a wiki's index page with categories, entity counts, last-updated dates, and navigational structure.

### The Compaction Wall

The deepest gap is architectural. Compaction — the most frequently-used persistence mechanism — is **destructive**. It throws away detail to save tokens. A wiki would do the opposite: compile detail into a persistent artifact where it accumulates and becomes more valuable over time. Every time Claude Code compacts a conversation, knowledge moves from a rich representation (the full message history) to a poor one (a paragraph summary). The information exists in the transcript on disk, but it's effectively inaccessible — buried in megabytes of unindexed JSON.

---

## The Complete Pipeline

Here's how knowledge flows through Claude Code's persistence layers:

1. **Session starts**: Load CLAUDE.md stack (managed → user → project → local). Load MEMORY.md into system prompt. Topic files available on demand.

2. **During session**: Model reads files, runs commands, generates insights. All stored in the context window (working memory). Nothing persists yet.

3. **Context fills**: Compaction fires. Old messages are summarized into a boundary message. Detail is lost. Discovered tools are preserved as metadata.

4. **Turn ends**: Memory extraction agent (if enabled) forks from the main conversation. Scans the transcript for durable knowledge. Writes to topic files in the memory directory. Updates MEMORY.md index.

5. **User says "remember this"**: Model writes directly to memory files. Extraction agent skips this turn to avoid duplication.

6. **Session ends**: Full transcript written to JSONL file. Compacted summaries included. Raw tool outputs preserved.

7. **Next session starts**: MEMORY.md loaded (200 lines max). CLAUDE.md loaded. Previous session's transcript available via grep but not automatically loaded. Everything not in memory or CLAUDE.md must be re-derived.

The persistence architecture is conservative by design. It saves little, loads little, and trusts the model to re-derive what it needs from the codebase. This works because codebases are their own knowledge base — the model can always re-read the source. What it can't re-derive is the user's preferences, the project's conventions, the lessons from debugging sessions, and the strategic context behind decisions. Those are what the memory system is for, and those are what fall through the gaps when the extraction agent doesn't run, the user doesn't say "remember this," and compaction throws away the details.

The seed of a wiki is here: a persistent directory of typed markdown files with an index entrypoint, a typed taxonomy of memory categories, a background agent that extracts knowledge without interrupting the main task, and a daily-log mode that acknowledges real-time synthesis is too expensive for long sessions.

But the compounding property — where every interaction makes the knowledge base richer, where cross-references build automatically, where contradictions are flagged, where insights are filed back — that's not implemented yet. The KAIROS daily-log mode comes closest: append-only logging with nightly distillation is exactly the write-now-synthesize-later pattern the wiki needs. If that distillation step were generalized beyond daily logs to cover all session transcripts, and if the synthesis produced interlinked entity pages rather than flat topic files, the architecture would cross the threshold from memory storage to knowledge building.

The architecture stores memories. It doesn't build understanding. The gap between those two is the gap between a file system and a wiki — and that gap is where the most valuable knowledge falls through.
