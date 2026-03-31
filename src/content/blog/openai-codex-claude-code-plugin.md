---
title: "OpenAI Just Shipped a Plugin So Codex Runs Inside Claude Code"
description: "OpenAI open-sourced a plugin so Codex runs inside Claude Code. The real signal isn't the commands, it's the distribution strategy hiding in the architecture."
pubDate: 2026-03-31
tags: ["OpenAI", "Codex", "Claude Code", "AI Agents", "Developer Tools"]
---

## The real story isn't the three commands. It's the admission hiding inside the architecture.

---

I keep coming back to what OpenAI shipped on March 30 and 31. They put out `codex-plugin-cc`, open source under Apache 2.0, so Codex can run inside Claude Code. That caught me off guard a little. I could be wrong, but I can't remember another OpenAI move this direct into a rival dev surface.

On paper it's a tiny surface area. Actually, wait, that's not quite right. It's narrow, not small. `/codex:review` does the read-only pass. `/codex:adversarial-review` is the skeptical one that goes after tradeoffs and failure modes. `/codex:rescue` hands the work to a Codex subagent, and the repo also ships `/codex:status`, `/codex:result`, plus `/codex:cancel` for background jobs. Small thing, big signal.

## What it isn't

The part that makes this more interesting is what it isn't. At least from the repo, this doesn't read like a deep MCP-style bridge. It looks like Claude Code plugin plumbing: markdown command files, hooks for session start/end plus `Stop`, and a `codex-rescue` agent file. The command definitions literally tell Claude Code to run `node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" ...`, and the rescue agent is described as a thin forwarder that makes one Bash call.

Under the hood it's even more blunt, which I mean as a compliment. Slash command, subprocess, Node companion, then a shared broker talking JSON-RPC style messages over a Unix socket to the Codex app server. Fire, wait, print. The broker code handles methods like `turn/start` and `review/start`, and OpenAI's own README says the plugin wraps the Codex app server rather than spinning up some separate runtime.

No separate auth either. It rides the same local Codex CLI login and config you already have, which makes the whole thing feel closer to a sharp CLI wrapper than a native co-reasoning tool. `/codex:rescue` can detach into tracked background jobs, and there's even an optional stop-time review gate wired to Claude Code's `Stop` hook. That's clever, slightly chaotic, and honestly pretty useful.

## Distribution math

It also lands right as OpenAI is pushing Codex plugins much harder. Their docs now treat plugins as bundles that can mix skills with app integrations or MCP servers, and the examples already point at Slack and Linear, plus a Sentry-flavored workflow. So this Claude Code repo doesn't feel random to me. It feels like distribution math.

My read? OpenAI picked the faster path, not the deepest one. They got Codex inside a rival's editor without needing the cleanest possible protocol story. A fuller MCP route would've been more intimate. Claude could call Codex mid-loop and react to it on the fly. But that isn't what this repo is. This is closer to "shell out, let Codex cook, hand the text back," and that might be exactly why it shipped this week instead of later.

## The real signal

I don't think the real story is the three commands. I think it's the admission hiding inside the architecture: devs pick their own surface area, and the model company that shows up there wins more than the one that insists everyone come home.

---

*Follow me on [X](https://x.com/oldeucryptoboi) — I post as @oldeucryptoboi.*
