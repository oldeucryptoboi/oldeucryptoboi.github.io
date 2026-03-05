---
title: "Why I Built a Native macOS Client for My AI Agent"
description: "Why I built a native macOS/iOS client for my autonomous AI agent instead of staring at terminal output"
pubDate: 2026-03-05
tags: ["AI Agents", "KarnEvil9", "Tarkus", "Swift", "macOS", "Open Source"]
---

## Terminal output is fine until your agent asks for permission to delete a database

I've been running EDDIE — my autonomous AI agent — inside KarnEvil9 for months now. He operates on scheduled tasks, reviews code, posts to social media, files GitHub issues. The whole thing runs headless on a server, and I interact with it through the API or CLI.

That worked. Until it didn't.

The problem isn't running the agent. The problem is *watching* the agent. KarnEvil9's execution model is step-based: the planner generates a plan, each step runs through permission gates, tools get invoked, results come back, and sometimes the agent needs approval before it can proceed. All of that shows up as JSON events in a WebSocket stream or a JSONL journal file.

Reading raw journal events at 2am to figure out why Eddie decided to rewrite a PR description instead of posting it isn't great. Scrolling through terminal output to find the one approval request buried in fifty step events isn't great either. And when Eddie's futility monitor kicks in and halts execution, I want to know immediately, not whenever I happen to check the logs.

So I built Tarkus.

## What Tarkus actually is

Tarkus is a native Swift app for macOS and iOS. It connects to a KarnEvil9 server and gives you a chat interface to EDDIE. You type a task, EDDIE plans and executes it, and you watch the whole thing happen in real time.

But it's not just a chat wrapper. The interesting parts are the things you can't do in a terminal.

**Step activity shows up inline.** When EDDIE's executing a plan, each step appears as a card inside the conversation. You see the tool name, what it's doing, whether it succeeded or failed. If a step involves reading a file, you see the output. If it's running a shell command, you see what command and what came back. It's not a separate panel or a different screen — it's right there in the flow of the conversation.

**Approvals are interactive.** This is the one that made me build the whole thing. KarnEvil9's permission system means EDDIE regularly needs approval to use certain tools. In the terminal, that's a blocking prompt. In Tarkus, an approval card slides into the chat with the tool name, what it wants to do, and the input parameters. You tap Grant or Deny. You can grant temporarily or permanently. The agent continues.

**Bonjour discovery.** KarnEvil9 servers advertise themselves as `_karnevil9._tcp.` services. Tarkus finds them automatically on your local network. No typing IP addresses, no config files. Open the app, your server shows up, connect.

## The parts I didn't expect to matter

I spent way too long on markdown rendering. EDDIE's responses are full of code blocks, tables, links, lists. The first version used a SwiftUI markdown library and it was... fine. Then I rewrote it with cmark-gfm and Highlightr for syntax highlighting and suddenly the responses felt like reading actual documentation instead of a wall of monospace text.

The session history turned out to be more useful than I thought. I originally added it as a "nice to have" — a list of past sessions you can browse. But now I use it constantly to go back and check what EDDIE did on a scheduled task three days ago. Each session has its full journal, so you can replay the entire execution sequence: what was planned, what ran, what got approved, what failed.

Notifications changed how I work with EDDIE too. When an approval request comes in, I get a push notification on my phone. Tap it, Tarkus opens to the approval screen, I grant it, EDDIE continues. The whole interaction takes about four seconds. Before Tarkus, approval requests would just sit there until I happened to check.

## How it connects

Under the hood, Tarkus talks to KarnEvil9's REST API and WebSocket endpoint. The WebSocket handles real-time events — session created, step started, step completed, approval requested, session done. If the WebSocket drops, it falls back to Server-Sent Events. There's auto-reconnection with exponential backoff so it doesn't hammer the server.

The data model maps directly to KarnEvil9's event types. A `step.started` event creates a step card in the chat. A `step.completed` event updates that card with results. An `approval.requested` event adds an interactive approval card. A `session.completed` event marks the conversation as done.

Authentication uses a Bearer token stored in Keychain. Server config lives in UserDefaults. Nothing fancy.

## What's next

I'm working on a dashboard view with metrics — token usage per session, step execution timelines, approval response times. The data's all there in the journal, I just need to visualize it.

The other thing I want is multi-server support. Right now Tarkus connects to one KarnEvil9 server. But with the P2P delegation mesh in KarnEvil9's swarm plugin, you could have multiple agents running on different machines. Being able to monitor all of them from one app would be useful.

Tarkus is open source at [github.com/oldeucryptoboi/tarkus](https://github.com/oldeucryptoboi/tarkus). It requires a running KarnEvil9 server to connect to. If you're building on KarnEvil9, it's the easiest way to interact with your agents.
