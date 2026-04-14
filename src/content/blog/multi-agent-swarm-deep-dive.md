---
title: "How the Multi-Agent Swarm Actually Works"
description: "The Problem"
pubDate: 2026-04-14
tags: []
---

Claude Code can run multiple agents at the same time. A leader agent orchestrates workers that run in parallel, in separate terminal panes, in background processes, or in the same Node.js process. They coordinate through files on disk. Here is every mechanism, traced end-to-end through the source.

---

## The Problem

The simplest version of multi-agent coding is to run multiple CLI instances on the same repository and let them share a filesystem. Each agent works on its own task, reads and writes files, and eventually you merge the results. This approach fails almost immediately.

State collisions come first. Two agents editing the same file produce corrupted output. Even agents working on different files can collide: one agent installs a dependency while another is mid-build, and the build fails with a partial lockfile. There is no coordination layer to prevent this, so agents step on each other constantly.

Permission storms come next. Every agent independently asks the user for permission to run commands, read files, or access the network. With five agents running, the user faces a stream of interleaved permission prompts with no way to tell which agent is asking for what. The mental overhead makes the system unusable.

Then there is lifecycle management. If the user cancels the leader task, the worker processes keep running. They have no parent to report to, no signal to stop, and no cleanup logic. They become zombie processes that continue modifying files after the user thinks everything has stopped.

The real challenge has three parts. First, **isolation**: workers must not stomp each other's mutable state, UI callbacks, or permission tracking. Second, **communication**: the leader must be able to assign work, receive results, and relay permission decisions. Third, **lifecycle management**: workers must die when the leader dies, and cleanup must always run.

The design principle that solves all three is **uniform communication, pluggable execution**. All three execution modes (in-process, tmux panes, and iTerm2 panes) use the same file-based mailbox for coordination. The execution backend is swappable. The mailbox does not care which backend spawned the worker. A leader can have some workers running as in-process coroutines and others running in terminal panes, and the communication protocol is identical. This separation means the coordination logic is written once and tested once, while new execution backends can be added without touching the mailbox system.

The file-based mailbox is the key architectural decision. It could have been a TCP socket, a Unix domain socket, or shared memory. Files were chosen because they work across process boundaries (pane-based workers are separate processes), survive brief disconnections, provide a natural audit trail, and require no daemon process. The tradeoff is latency: file I/O is slower than IPC. But for a system where messages are human-readable task assignments and status updates, 5-100ms of lock contention is invisible.

---

## The Three Execution Modes

### In-Process: AsyncLocalStorage Isolation

The lightweight path. The leader and all workers share one Node.js process. No child processes, no IPC, no terminal panes. Workers are concurrent async tasks running in the same event loop.

The isolation mechanism is `AsyncLocalStorage`, a Node.js primitive that carries context through the async call stack without threading it through every function parameter. Each worker runs inside `AsyncLocalStorage.run()` with a `TeammateContext` that carries identity: name, team, color, and parent session ID. Any function anywhere in the call stack can call `getTeammateContext()` to discover "who am I?" without the identity being passed explicitly. This is critical because the codebase has hundreds of functions between the top-level agent loop and the low-level operations that need to know which agent is running.

#### Two-Level Abort Hierarchy

Each worker gets two abort controllers, not one. The first is a **lifecycle controller**: aborting it kills the worker entirely. This controller is deliberately independent from the leader's controller. Workers survive when the user interrupts the leader's current query; a leader interrupt should not kill workers mid-task.

The second is a **per-turn controller** created fresh at the start of each iteration of the worker's main loop. This controller is stored in the worker's task state so the UI can reach it. When the user presses Escape, it aborts only the per-turn controller, stopping the current API call and tool execution without killing the worker. The worker exits its current turn, sends an idle notification, and waits for its next instruction. The lifecycle controller remains untouched. The worker is still alive.

```
main while loop:
    create currentWorkAbortController        ← new each iteration
    store in task state for UI access
    run agent turn (uses currentWorkAbortController)
    if currentWorkAbortController.aborted:
        break out of agent turn, stay in while loop
    clear controller from task state
    send idle notification
    wait for next prompt or shutdown
```

This two-level scheme means Escape stops current work (fast feedback) without losing the worker (no re-spawn cost). Force-killing the lifecycle controller is reserved for shutdown and cleanup.

#### ToolUseContext Cloning

When the leader spawns a worker, it creates a subagent context by selectively cloning some fields and replacing others:

- **readFileState**: cloned. Workers cache file reads independently, so one worker's stale cache does not affect another.
- **setAppState**: replaced with a no-op. Workers cannot mutate the leader's UI state. Without this, a worker could overwrite the leader's status display, progress indicators, or tool output panels.
- **setAppStateForTasks**: shared, pointing at the root store. This is the critical exception to the isolation rule. When a worker spawns a background bash command, that command must be registered in the root application state. If it were registered in a no-op store, the command would become an orphan zombie process: no parent tracking it, no cleanup killing it. Safety over purity.
- **contentReplacementState**: cloned (not fresh). A clone makes identical replacement decisions as the parent, which keeps the API request prefix byte-identical and preserves prompt cache hits. A fresh state would diverge and bust the cache.
- **localDenialTracking**: fresh. The denial counter (which tracks how many times a user has denied a particular permission) must accumulate per worker, not per process. Otherwise one worker's denied permissions would affect another worker's escalation behavior.
- **UI callbacks** (setToolJSX, addNotification): set to undefined. Workers have no UI surface.
- **shouldAvoidPermissionPrompts**: set to true. Workers must never prompt the user directly; they escalate to the leader.

The leader passes `messages: []` to the worker. The worker never sees the leader's conversation history. It receives only its initial prompt: the task description written by the leader. This is both an isolation measure (workers should not reason about the leader's full context) and a practical one (the leader's context window is already large; duplicating it per worker would be wasteful).

#### Team-Essential Tool Injection

Even when a worker is configured with an explicit tool list (e.g., only file-reading tools), seven tools are always injected: SendMessage, TeamCreate, TeamDelete, TaskCreate, TaskGet, TaskList, TaskUpdate. Without these, a worker receiving a shutdown request could not acknowledge it (no SendMessage), and a worker assigned tasks from the task list could not update them. The injection uses set-deduplication so tools already in the list are not duplicated.

### Pane-Based: tmux and iTerm2

The visual path. Each worker is a separate Claude Code process running in a visible terminal pane. The user can watch workers in real time, see their output, and even type into their panes. This mode exists because observability matters. For complex multi-agent tasks, watching the workers is more informative than reading their final summaries.

**tmux mode** has two sub-cases depending on whether the leader is already inside a tmux session.

If the leader is inside tmux, it splits its own window: 30% on the left for the leader, 70% on the right for workers. Workers stack vertically on the right side. This keeps the leader visible while giving workers most of the screen real estate.

If the leader is outside tmux, it creates a standalone tmux session named `claude-swarm` on a separate socket. Workers tile inside this session. The separate socket prevents collision with the user's existing tmux sessions.

Pane creation is serialized through an async lock, implemented as promise chaining, not a mutex. Without this lock, concurrent `tmux split-pane` calls race against each other and produce incorrect layouts. tmux's internal state is not safe for concurrent modification, so each pane creation must complete before the next one starts. A 200ms shell initialization delay between spawns ensures the pane's shell is ready before the Claude Code command is sent to it.

**The ORIGINAL_USER_TMUX problem.** Detection of whether the user started Claude from inside tmux must capture the `TMUX` environment variable at module load time. Later during startup, the shell module overrides `TMUX` when Claude's own internal tmux socket is initialized. Without the early capture, the detection function would always think it is inside tmux: it would see Claude's own socket, not the user's original session. A separate capture of `TMUX_PANE` preserves the leader's original pane ID for the same reason.

**iTerm2 mode** uses the `it2` CLI, a Python API wrapper for iTerm2's scripting interface. The first worker splits vertically from the leader's session. Subsequent workers split horizontally from the last worker, producing a horizontal stack. Dead session recovery prunes disappeared UUIDs and retries with the next-to-last worker, or falls back to the leader's UUID. This retry is bounded at O(N+1) attempts.

**Detection priority** determines which mode is used when the user has not specified one: tmux (if already inside) > tmux (if available on PATH) > iTerm2 (if available) > in-process (always available). The detection runs once at startup and caches the result. The preference for tmux-inside over tmux-available reflects a UX judgment: if the user is already in tmux, panes should appear in their existing session rather than creating a disconnected one.

**Sticky fallback.** Once the in-process fallback is activated (e.g., because tmux and iTerm2 are both unavailable), it stays active for the entire session. This prevents oscillation. If the detection environment has not changed, re-running detection would produce the same result, so the system caches the fallback decision permanently.

### Fork Subagents

The fork subagent variant is fundamentally different from normal subagents. A normal subagent starts with an empty message history and only its task prompt. A fork subagent inherits the parent's **entire message history and system prompt byte-for-byte**. This maximizes prompt cache hits. The API caches based on prefix matching, so if five fork children share the same message prefix (the parent's full history), only the first child incurs the full input cost.

The critical mechanism is **renderedSystemPrompt threading**. The parent does not tell the fork to re-build its own system prompt by calling the system prompt generator. Re-calling the generator can produce subtly different bytes because feature flags may have warmed up since the parent's prompt was built. A single bit of divergence busts the cache prefix entirely. Instead, the parent passes its already-rendered system prompt bytes through a shared parameter object. The fork uses those exact bytes, guaranteeing a byte-identical prefix.

Each fork child's message history is constructed to be cache-identical through the shared prefix. The parent's tool results are replaced with placeholder blocks (preserving byte positions), and each child receives its specific task as the final text block. Everything before that final block is identical across siblings.

Fork guards prevent infinite recursion through two levels:

1. **Primary**: the query source field. If it indicates a fork origin, the agent cannot re-fork.
2. **Secondary**: a scan of the message history for a fork boilerplate tag. This guard survives context compaction. Even if the system compresses earlier messages, the tag persists in the remaining history.
3. **Explicit instruction**: fork children are told "Do NOT spawn sub-agents. Execute directly."

---

## The Mailbox System

Every agent, regardless of execution mode, has a JSON inbox file on disk. Communication between agents is message passing through these files, serialized by file-level advisory locks.

### Path Structure

```
~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

Each team gets its own directory. Each agent within the team gets a single inbox file. The inbox is a JSON array of messages.

### Write Protocol

Writing a message to another agent's inbox follows a careful protocol to prevent data loss:

```
function write_to_mailbox(recipient, message):
    ensure inbox directory exists
    create inbox file atomically (exclusive-create, exists-ok)
    acquire advisory lock (retry 10x, backoff 5-100ms exponential)
    re-read messages from file
    append new message with read=false
    write updated array back to file
    release lock
```

The critical step is the **re-read after lock acquisition**. Without it, two concurrent writers would both read the inbox before either acquires the lock. Writer A acquires, appends its message, writes. Writer B acquires, appends its message to the **stale** copy it read before the lock, writes, overwriting Writer A's message. By re-reading inside the lock, Writer B sees Writer A's message and appends to the current state.

The advisory lock uses 10 retries with 5ms minimum and 100ms maximum exponential backoff. This is sized for approximately 10 concurrent agents. The fast path acquires in under 5ms; the worst case retries 10 times before failing. The bound is finite and will not hang indefinitely.

### Read Protocol

Reading follows the same locking discipline. The recipient acquires the advisory lock, reads its inbox file, filters for unread messages, processes them, marks them as read, and writes the updated array back. The same lock protects the read-modify-write cycle.

### Clearing and Fail-Closed Semantics

The `clearMailbox` function opens the file with a flag that requires the file to already exist. If the inbox does not exist (no messages have ever been sent), the open fails silently rather than creating an empty file. This prevents a subtle bug where clearing a nonexistent inbox would create an empty file, which other code might interpret as "inbox exists, agent is active."

The `readMailbox` function returns an empty array on ENOENT (no crash on a missing inbox). The `writeToMailbox` function treats EEXIST on file creation as silently ok. These are fail-closed boundaries: no operation creates phantom state, and missing state is treated as empty, not as error.

### Why Files?

The file-based approach has tradeoffs. It is slower than shared memory or Unix sockets. It requires lock management. It creates filesystem artifacts that need cleanup.

But it has properties that matter for this system: it works across process boundaries without IPC setup, it is inspectable by users and agents, it survives brief crashes (the inbox persists on disk), and it requires no daemon process. The filesystem is the message broker.

---

## Structured Protocol Messages

The mailbox carries both free-text messages (task assignments, status updates, questions between agents) and structured protocol messages that drive the coordination machinery. A type-checking function gates them: structured messages are dispatched to specific handlers, never fed to the language model as conversation input. If a `shutdown_request` JSON blob appeared in the model's history, it might try to "respond" conversationally or generate text that mimics the protocol format.

### Shutdown Protocol

Shutdown uses a three-message handshake:

```
leader -> worker:  shutdown_request  { requestId, reason }
worker -> leader:  shutdown_approved { requestId, paneId, backendType }
              OR
worker -> leader:  shutdown_rejected { requestId, reason }
```

A worker in the middle of a critical operation (mid-file-write, mid-git-commit) can reject the shutdown and finish its work. The `requestId` ties the response to the request, preventing a stale response from a previous attempt from matching a new one.

Force-kill bypasses the handshake entirely: abort the worker's lifecycle controller (in-process), kill the pane (tmux), or close the session (iTerm2).

### Permission Escalation

When a worker encounters an operation that requires user permission, it cannot prompt the user directly. The permission must be escalated to the leader. The escalation has two paths and a preliminary classifier step.

#### Bash Classifier Pre-Check

Before escalating, in-process workers try the bash classifier for auto-approval on bash commands. The worker **awaits** the classifier result. It does not race it against user interaction the way the main agent does. The main agent shows a permission prompt while the classifier runs in the background, accepting whichever resolves first. Workers cannot show prompts, so they wait for the classifier's verdict. If the classifier approves, the tool executes immediately with no leader involvement. If it does not approve, the worker falls through to escalation.

This is a latency-for-safety tradeoff specific to workers. The main agent races because it has a UI and can show a prompt while the classifier thinks. Workers have no UI, so racing would mean escalating to the leader while a classifier approval is still in flight, which would show the user a prompt that auto-resolves moments later. Awaiting avoids this confusing UX.

#### In-Process Fast Path

The worker writes to the leader's `ToolUseConfirmQueue`, an in-memory data structure shared within the process. The entry includes the tool name, input, and a `workerBadge` with the worker's name and color. The leader's UI picks up the queued request and renders a colored badge identifying which worker is asking. The user sees something like "[researcher] wants to run: npm install lodash" and can approve or deny. Sub-millisecond latency since it is just a shared memory write.

The entry also carries a `recheckPermission` callback. While the permission prompt is showing, conditions may change: the bash classifier might finish, or a team-wide permission broadcast might grant the needed access. The UI periodically calls `recheckPermission` to check if the prompt can auto-resolve without user input.

#### Mailbox Fallback Path

For pane-based workers (separate processes), the in-memory queue is not available. The escalation follows a longer path:

```
worker: createPermissionRequest(tool, input)
     -> registerPermissionCallback({ requestId, onAllow, onReject })
     -> sendPermissionRequestViaMailbox(leaderInbox, request)
     -> start polling own mailbox at 500ms intervals

leader: inbox poller detects permission_request
     -> renders PermissionRequest UI with WorkerBadge
     -> user approves or denies
     -> sendPermissionResponseViaMailbox(workerInbox, response)

worker: poll finds permission_response
     -> processMailboxPermissionResponse()
     -> fires registered callback (onAllow or onReject)
     -> tool executes or returns denial
```

The registered callback pattern decouples the mailbox polling loop from the specific permission request. Multiple permission requests from different tool calls can be in flight simultaneously, each with its own callback.

#### Permission Persistence

Permission updates (the allow-rules the user creates when they say "always allow this") are persisted to the leader's permission context with a `preserveMode` flag. This flag ensures the worker's restricted mode does not widen the leader's mode. If a worker is running in a more restricted permission mode and the user approves a specific tool for that worker, the approval is scoped. Without `preserveMode`, the worker's mode could leak upward and relax the leader's security posture.

### Other Protocol Messages

**Plan approval**: workers in plan mode send the plan file path and content; the leader presents it to the user and responds with approval, optional feedback, and the execution permission mode.

**Sandbox network permissions**: when a sandboxed worker's code attempts to reach a non-allowlisted host, the sandbox escalates to the leader with the host pattern.

**Task assignment**: carries task IDs from the shared task system, allowing the leader to assign specific tasks to specific workers.

**Mode control**: allows the leader to remotely change a worker's permission mode, for example upgrading from plan mode to full execution after approving the plan.

**Team permission broadcast**: when one worker gets permission to access a directory, that permission is broadcast to all workers on the team, preventing the user from approving the same directory for every worker individually.

---

## Git Worktree Isolation

File-level isolation prevents collisions for mutable runtime state, but it does not solve the fundamental problem of multiple agents editing the same repository. Two agents modifying different functions in the same file produce a merge conflict. Two agents running tests concurrently interfere with each other's build artifacts. Git worktrees solve this.

### Creation with Path Traversal Protection

When an agent is spawned with worktree isolation, the slug is validated before any filesystem operation. Each slash-separated segment must match `[a-zA-Z0-9._-]+`, and the literal segments `.` and `..` are rejected. The total length is capped at 64 characters. Without this validation, a slug like `../../../etc` would escape the worktrees directory via `path.join` normalization and create a worktree anywhere on the filesystem.

Symlink targets are also validated. Before creating a symlink from the worktree to the main repository, the system checks for path traversal in the target, preventing a malicious symlink target from pointing outside the repository.

```
function create_agent_worktree(slug):
    validate slug (per-segment regex, reject . and .., max 64 chars)

    if WorktreeCreate hook exists:
        delegate to hook (VCS-agnostic)
        return

    worktree_path = {repo}/.claude/worktrees/{slug}/
    branch = "claude-wt-{timestamp}-{slug}"
    git worktree add {worktree_path} -b {branch}

    post-creation setup:
        copy settings.local.json
        configure git hooks (symlink .husky or .git/hooks)
        symlink large directories (node_modules, .next)
        copy .worktreeinclude files
```

### The .worktreeinclude Mechanism

Some files are gitignored but essential for the project to function: environment files, generated configuration, binary assets. A plain git worktree does not include these because git does not track them.

The `.worktreeinclude` file (in the repository root, using gitignore-style pattern syntax) lists patterns for files that should be copied to worktrees. The copy logic requires files to match BOTH conditions: listed in `.worktreeinclude` AND gitignored. Files that are tracked by git are already in the worktree via the checkout; this mechanism only handles the gitignored gap.

The implementation uses `git ls-files --directory` to efficiently list gitignored paths, collapsing fully-ignored directories into single entries rather than enumerating every file inside them. When a pattern targets a path inside a collapsed directory, the system expands that specific directory with a scoped `ls-files` call.

### Symlink Optimization

Multiple concurrent worktrees can consume significant disk space. The `node_modules` directory alone might be hundreds of megabytes. Multiply by five workers and the cost is gigabytes of duplicated dependencies.

Directories listed in the worktree symlink configuration (e.g., `node_modules`, `.next`) are symlinked from the worktree back to the main repository rather than copied. All worktrees share the same physical directory. The tradeoff: a worker installing a new dependency affects all other workers. In practice workers rarely modify dependencies. They edit source code.

### Cleanup: Fail-Closed

```
function cleanup_worktree(info):
    if hook-based:
        keep (cannot detect VCS changes generically)
    if has_uncommitted_changes(worktree, headCommit):
        keep worktree
    else:
        git worktree remove --force
        git branch -D {branch}
```

The change detection check is **fail-closed**: if `git status` fails, if `git rev-list` fails, or if any other error occurs, the function returns true ("yes, there are changes, keep the worktree"). The cost of keeping an empty worktree is a few megabytes. The cost of deleting a worktree with the user's changes is catastrophic.

### Fork Subagents with Worktrees

When a fork subagent runs in a worktree, it inherits the parent's message history, which contains file paths from the parent's working directory. A `worktreeNotice` is injected:

> "You've inherited context from a parent at {parentCwd}. You're in an isolated worktree at {worktreeCwd}. Translate paths. Re-read files before editing, the worktree may have diverged."

---

## The Idle Loop and Context Management

After a worker completes its current task, it enters an idle loop that polls the mailbox for new instructions. This loop is where message priority, compaction, and task claiming happen.

### Message Priority

The idle loop reads all unread messages and applies a strict priority order:

1. **Shutdown requests**: scanned first across all unread messages. A shutdown request buried behind ten peer messages is still processed immediately.
2. **Team-lead messages**: the leader represents user intent and coordination. Its messages should not be starved behind peer-to-peer chatter.
3. **FIFO peer messages**: messages from other workers, processed in arrival order.
4. **Unclaimed tasks**: if no messages are waiting, the worker checks the shared task list for available work and claims the next item.

This priority order prevents starvation. Without it, a flood of peer-to-peer messages could delay a shutdown request indefinitely, leaving a zombie worker running after the user thinks everything has stopped.

### Compaction Within the Teammate Loop

Workers have their own conversation history that grows with each turn. When the token count (estimated, not exact) exceeds the auto-compact threshold, the worker runs `compactConversation`, the same compaction logic the main agent uses. This creates an isolated copy of the ToolUseContext for compaction, then resets the microcompact state and content replacement state afterward.

Without this, a long-running worker would eventually exceed its context window and fail. The compaction keeps the worker's history bounded while preserving the essential information from earlier turns.

### Idle Notification

When a worker finishes a turn and enters the idle loop, it sends an `idle_notification` to the leader's mailbox:

- **idleReason**: 'available' (finished successfully), 'interrupted' (user pressed Escape), or 'failed' (error occurred).
- **summary**: a 5-10 word summary extracted from the worker's most recent SendMessage tool use. Lets the leader understand what each worker accomplished without reading the worker's full output.
- **completedTaskId** and **completedStatus**: for task-aware coordination, allowing the leader to update the shared task list.

---

## Lifecycle and Cleanup

Every execution mode has a cleanup chain that ensures workers do not outlive their leader, zombie processes do not accumulate, and resources are released.

### In-Process Cleanup

```
on leader exit:
    registerCleanup -> abort all worker lifecycle AbortControllers

on worker completion:
    invoke and clear onIdleCallbacks
    send idle_notification to leader mailbox
    update AppState task status
    unregister Perfetto tracing agent

on worker kill:
    abort lifecycle controller
    alreadyTerminal guard: check if status != 'running'
        if already killed/completed, skip (prevents double SDK bookend)
    update task status to 'killed'
    remove from teammates list
    evict task output from disk
    emit SDK task_terminated event
```

The **alreadyTerminal guard** prevents a race between natural completion and forced kill. If a worker finishes its task and sets its status to "completed" at the same moment the leader sends a kill, the kill handler would find a non-running status and skip the status update. Without this guard, the SDK would emit two lifecycle bookend events for the same worker, confusing any tooling consuming the event stream.

### Pane-Based Cleanup

```
on leader exit:
    registerCleanup -> Promise.allSettled(kill all panes)

on pane close:
    worker process exits naturally (stdin closed)
    leader detects via is_active check on next poll
```

Pane cleanup uses `Promise.allSettled`, not `Promise.all`. If one pane kill fails (the user already closed it manually, or the tmux server crashed), the remaining panes are still killed. `Promise.all` would short-circuit on the first failure and leave surviving panes as zombies.

For tmux, the leader polls pane liveness by checking whether the pane target still exists. For iTerm2, the leader checks session UUIDs. A disappeared pane means the worker is dead. No ambiguity, no zombie state.

### Cleanup Registration

Both execution modes register their cleanup functions at the point of worker creation, not at the point of leader exit. This ensures cleanup runs even if the leader crashes unexpectedly. The cleanup registry is invoked on process exit, signal handlers (SIGINT, SIGTERM), and uncaught exception handlers.

### The Zombie Prevention Invariant

The `setAppStateForTasks` punch-through is the most important cleanup invariant. When a worker spawns a background bash command, that command runs as a child process that must be registered in the root application state for tracking and cleanup.

For in-process workers, `setAppState` is a no-op. Workers cannot mutate the leader's UI. If `setAppStateForTasks` were also a no-op, the bash command would be spawned but never registered. When the session ends, the command would still be running. Its parent PID becomes 1 (init/launchd), making it an untracked zombie.

The punch-through points directly at the root store. Every background command is registered regardless of which agent spawned it. This is an explicit choice of safety over purity: a cleaner isolation model would fully isolate workers from the root store, but the consequence (zombies) is worse than the consequence of partial isolation.

---

## The Full Round-Trip

Here is every function in the path from the user invoking the Task tool to a worker requesting and receiving permission for a bash command. This is the in-process execution mode.

```
User invokes Task tool with agent configuration
-> AgentTool handler: spawnTeammate(config, toolUseContext)
-> spawnMultiAgent: route to handleSpawnInProcess()
-> spawnInProcess:
    create TeammateContext (AsyncLocalStorage container)
    create independent lifecycle AbortController
    register task state in AppState
    register cleanup handler
-> InProcessBackend.spawn() -> startInProcessTeammate()
-> runInProcessTeammate() [fire-and-forget]:
    create AgentContext (for analytics)
    build system prompt (default + teammate addendum + custom agent prompt)
    enter main while loop:
        create per-turn currentWorkAbortController
        store in task state
        runWithTeammateContext -> runWithAgentContext -> runAgent:
            query(): core API call
                model returns tool_use blocks
                runTools(): partition tool calls into concurrent/serial batches
                runToolUse():
                    call canUseTool (from createInProcessCanUseTool)
                    hasPermissionsToUseTool() returns 'ask'
                    [CLASSIFIER] if bash command and classifier enabled:
                        await classifier verdict (not race)
                        if approved: return allow, skip escalation
                    [FAST PATH] if leader bridge available:
                        push to ToolUseConfirmQueue with workerBadge
                        leader UI renders permission prompt
                        user approves -> onAllow fires
                        persistPermissionUpdates with preserveMode:true
                        return allow
                    [MAILBOX PATH] if bridge unavailable:
                        createPermissionRequest
                        registerPermissionCallback(requestId, onAllow, onReject)
                        sendPermissionRequestViaMailbox
                        poll own mailbox at 500ms
                        leader detects request, shows prompt
                        leader responds via mailbox
                        poll finds response -> processMailboxPermissionResponse
                        callback fires -> return allow or deny
                    tool.handler(input) executes
                response streamed back
        check compaction threshold -> compact if needed
        clear currentWorkAbortController from task state
    send idle_notification to leader mailbox
    waitForNextPromptOrShutdown():
        poll mailbox every 500ms
        priority: shutdown > team-lead > FIFO peers > unclaimed tasks
        return WaitResult
    on shutdown_request: pass to model (approveShutdown/rejectShutdown tool)
    on new_message: wrap in XML, loop back
    on abort: exit
    on exit: alreadyTerminal guard, update status, emit SDK event, evict output
```

---

## Design Trade-Offs

Six deliberate design trade-offs, each choosing one property over another:

**Safety over purity.** `setAppState` is a no-op for workers, but `setAppStateForTasks` punches through to the root store. Full isolation would be cleaner. Zombie prevention is more important.

**Safety over convenience.** Independent lifecycle AbortControllers per worker. Linking them to the leader's controller would be simpler. Workers surviving leader interrupts is more important.

**Latency over correctness.** tmux pane creation serialized with a 200ms delay between spawns. Parallel creation would be faster. Correct pane layouts are more important.

**Safety over disk.** `hasWorktreeChanges` is fail-closed. Any error keeps the worktree. Cleaning up empties would save disk. Never deleting user work is more important.

**Cache over isolation.** `contentReplacementState` is cloned, not fresh. Cloning makes the fork's API request prefix byte-identical to the parent, preserving prompt cache hits. A fresh state would be more isolated but would diverge and bust the cache.

**Safety over mode leakage.** Permission updates from workers use `preserveMode: true`. A worker running in a restricted mode cannot widen the leader's permission mode when its tool approvals are persisted. Without this flag, approving a tool for a restricted worker would relax the leader's security posture.

---

## Fail-Closed Boundaries

Every external interaction has a fail-closed boundary:

| Operation | Failure | Response |
|-----------|---------|----------|
| readMailbox | ENOENT | Return empty array |
| writeToMailbox | EEXIST on create | Silently ok |
| clearMailbox | ENOENT | Silently fail (no phantom inbox) |
| hasWorktreeChanges | Any git error | Return true (keep worktree) |
| isStructuredProtocolMessage | Parse failure | Return false (treat as free text) |
| isInsideTmux | Shell module overrides env | Uses captured ORIGINAL_USER_TMUX |
| isIt2CliAvailable | Version check passes when API disabled | Uses `session list` not `--version` |
| Lock acquisition | 10 retries exhausted | Fail (finite, no hang) |
| Pane cleanup | One pane kill fails | Promise.allSettled continues others |
| Worker status update | Already terminal | Skip (no double bookend) |

No failure mode creates phantom state, hangs indefinitely, or silently loses data. The system is designed so that the worst case of any single failure is a slightly degraded experience: an extra worktree on disk, a protocol message treated as text, a slower detection path. Never data loss or zombie processes.
