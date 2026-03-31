---
title: "Inside Claude Code's Architecture: The Agentic Loop That Codes For You"
description: "How Anthropic built a terminal AI that reads, writes, executes, asks permission, and loops until the job is done"
pubDate: 2026-03-31
tags: ["Claude Code", "Anthropic", "AI Agents", "Developer Tools", "Architecture"]
---

## How Anthropic built a terminal AI that reads, writes, executes, asks permission, and loops until the job is done

---

I've been living inside Claude Code for months. It writes my code, runs my tests, commits my changes, reviews my PRs. At some point I stopped thinking of it as a tool and started thinking of it as a collaborator with terminal access.

So I read the architecture doc. Not the marketing page, not the changelog — the actual internal architecture of how Claude Code works under the hood. And it's more interesting than I expected, because the design decisions explain a lot of the behavior I've been experiencing as a user.

Here's what's actually going on.

## The agentic loop

Claude Code isn't a chatbot with a code plugin. It's an agentic loop.

You type something. Claude responds with text, tool calls, or both. Tools execute with permission checks. Results feed back to Claude. Claude decides whether to call more tools or respond. Loop continues until Claude produces a final text response with no tool calls.

That's it. That's the whole thing. But the details matter.

The loop is streaming-first. API responses come as Server-Sent Events and render incrementally. Tool calls are detected mid-stream and trigger execution pipelines before the full response is even done. This is why Claude Code feels responsive even when it's doing complex multi-step work — you see thinking and tool calls appearing in real time, not after a long pause.

Claude can chain multiple tool calls per turn. That's why you'll sometimes see it read three files, run a grep, and edit a function all in one burst. It's not making separate requests for each — it's one API call that returns multiple tool_use blocks, each executing in sequence.

## The tool system

There are about 26 built-in tools. Each one implements the same interface:

- An input schema (validated with Zod before execution)
- A permission check (returns allow, deny, or ask)
- The actual execution logic
- UI renderers for the terminal display

The core tools are what you'd expect: `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`. These are the workhorses. But the meta tools are where it gets interesting.

`Task` spawns subagents — child conversations with Claude that get their own isolated context, execute tools, and return a summary. This is how Claude Code parallelizes work. When it needs to research something in one part of the codebase while editing another, it doesn't do them sequentially. It spawns a subagent for the research and continues editing in the main conversation.

MCP servers contribute additional tools at runtime. Your project can define custom tools — database queries, API calls, deployment scripts — and Claude Code picks them up automatically. The tools show up in Claude's palette alongside the built-in ones.

## Permissions: the part that actually matters

Five permission modes: default (ask for everything), acceptEdits (auto-approve file changes, ask for shell commands), plan (read-only until you approve), bypassPermissions (auto-approve everything), and auto (automation-friendly minimal approval).

But the modes are just the top layer. Every tool call goes through a five-step gauntlet:

1. The tool's own `checkPermissions()` — Bash checks for destructive commands, Write checks file paths
2. Settings allowlist/denylist — glob patterns like `Bash(npm:*)` or `Read(~/project/**)`
3. Sandbox policy — managed restrictions on paths, commands, network access
4. The active permission mode — may auto-approve or force-ask regardless of the above
5. Hook overrides — `PreToolUse` hooks can approve, block, or modify the call before it executes

This layered model is why Claude Code can feel both powerful and safe at the same time. When I'm in acceptEdits mode, it flies through file changes without asking. But if it tries to run `rm -rf` or push to main, the tool-level check catches it before the mode override even matters.

The hooks are the escape hatch for everything else. You can write a shell script that runs before every Bash command and blocks anything matching a pattern. You can run a linter after every file edit. You can inject additional context into every user prompt. It's event-driven and configurable in `settings.json`.

## Configuration hierarchy

Settings merge in a specific order, with later values winning:

```
Defaults → ~/.claude/settings.json (user global) → .claude/settings.json (project, checked into VCS) → .claude/settings.local.json (project local, gitignored) → CLI flags → environment variables
```

This is a good design. Your team checks in project-level settings (allowed tools, MCP servers, hooks). You override locally with your preferences. CI overrides with environment variables. Nobody steps on anyone else.

## Context management

Conversations persist across turns in `~/.claude/sessions/`. When you're approaching the context window limit, older messages get summarized — Claude Code calls this "context compaction." There are even pre/post hooks for the compaction step so you can preserve specific information that shouldn't get summarized away.

The memory system is layered too. CLAUDE.md files provide persistent instructions per-project. Auto-memory files in `~/.claude/memory/` accumulate patterns across sessions. Session history lets you resume or fork previous conversations.

This is the part that makes Claude Code feel like it "knows" your project. It's not magic — it's a well-designed context injection pipeline. CLAUDE.md gets loaded into every system prompt. Memory files get loaded on startup. Your conversation history from yesterday is still there when you `/resume`.

## Multi-agent coordination

Subagents via the `Task` tool run as nested conversations within the same process. Same Claude model, separate context window, returns a summary when done.

But there's also a Teams system that uses tmux for true parallelism. A lead agent creates a team, members get separate tmux panes with their own Claude sessions, and they communicate through a shared message bus. Each member gets role-specific instructions and tool access.

I haven't used Teams yet, but the architecture makes sense. Subagents are for quick parallel research within a single task. Teams are for genuinely parallel workstreams — one agent refactoring the backend while another updates the frontend tests.

## The React terminal

This one surprised me. The terminal interface is a React app rendered via [Ink](https://github.com/vadimdemedes/ink) — a React renderer for CLIs. The conversation view, input area, tool call displays, permission dialogs, progress indicators — all React components using Yoga (CSS flexbox) for layout and ANSI escape codes for styling.

It supports inline images via the iTerm protocol. Thinking blocks are collapsible. Tool results show previews with execution status. It's genuinely well-built terminal UI, not just `console.log` with colors.

## What the architecture tells you about the product

The interesting thing about reading an architecture doc isn't the individual components — it's the design priorities they reveal.

**Streaming-first** means they optimized for perceived speed over simplicity. SSE parsing mid-stream is more complex than waiting for a complete response, but it makes the tool feel alive.

**Hook-extensible everything** means they expect power users to customize aggressively. Nearly every action has a pre/post hook point. This isn't an afterthought — it's a core architectural decision.

**Layered permissions** means they took safety seriously without making it annoying. Five layers of checks sounds heavy, but in practice most tool calls resolve instantly because the mode and allowlist handle the common cases. The user only sees a prompt when something genuinely unusual happens.

**Single-process subagents, multi-process teams** means they thought carefully about the tradeoff between simplicity and parallelism. Subagents are lightweight and fast because they share a process. Teams are heavier but truly parallel because they run in separate tmux panes.

Claude Code isn't a chat wrapper around an API. It's an agent runtime with a terminal UI. The agentic loop, tool system, permission model, and hook architecture form a coherent system designed to let an LLM operate autonomously on your codebase while giving you exactly the control points you need to stay in charge.

That's the part that matters. Not what Claude Code can do — but how much thought went into making sure you can control what it does.

---

*The full architecture document is available in the [Claude Code repository](https://github.com/anthropics/claude-code).*

*Follow me on [X](https://x.com/oldeucryptoboi) — I post as @oldeucryptoboi.*
