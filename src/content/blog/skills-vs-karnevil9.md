---
title: "Anthropic Just Validated What KarnEvil9 Has Been Building"
description: "Agent Skills are a step toward structured execution. KarnEvil9 is the full staircase."
pubDate: 2026-02-27
heroImage: "../../assets/hero-skills-vs-karnevil9.png"
tags: ["AI Agents", "KarnEvil9", "Anthropic", "Agent Skills"]
---

## Agent Skills are a step toward structured execution. KarnEvil9 is the full staircase.

---

Anthropic released [Agent Skills](https://agentskills.io) as an open standard. Cursor adopted it. VS Code adopted it. Gemini CLI, OpenAI Codex, Roo Code, JetBrains Junie — over 30 agent products now support the same skill format. A SKILL.md file with YAML frontmatter, a description that doubles as a trigger condition, and an `allowed-tools` field that restricts what the agent can touch.

This is significant. Not because of what Skills can do today, but because of what Anthropic is saying by releasing them: **agents need structured execution design, not just prompts.**

That's exactly what [KarnEvil9](https://github.com/oldeucryptoboi/KarnEvil9) has been building.

---

## What Skills Got Right

Skills solve a real problem. Agents are capable but context-starved. They don't know your team's deployment process, your code review standards, or your data pipeline quirks. Skills package that context into portable, version-controlled folders that agents load on demand.

The design has three smart ideas:

**Trigger conditions via description.** The `description` field in SKILL.md frontmatter isn't just documentation — it's the mechanism the agent uses to decide when to activate the skill. Include phrases like "sprint planning" or "design handoff" and the agent matches user intent to skill activation. Simple, elegant, zero configuration.

**Progressive disclosure.** Skills load in layers. First, the agent reads only the YAML frontmatter — enough to know when a skill is relevant. Then the full SKILL.md body. Then bundled scripts and references. Token-efficient by design. The agent doesn't pay for context it doesn't need.

**Tool restriction.** The `allowed-tools` field limits what a skill can access. A data analysis skill can run Python but not shell commands. A documentation skill can fetch URLs but not write files. This is access control at the skill level — coarse-grained, but real.

These are good engineering decisions. They show that Anthropic is thinking about agents as systems, not just language models with tool access.

---

## What Skills Don't Have

Skills are capability packages. They tell an agent *what* to do and *when* to activate. They don't govern *how* the agent executes, or *what happens when things go wrong*.

Here's what's missing — and what KarnEvil9 has:

**Permission gates.** Skills have `allowed-tools`, a flat list of tool names. KarnEvil9 has a multi-level permission engine with `domain:action:target` granularity. `filesystem:write:workspace` is a different permission than `filesystem:write:/etc`. Six decision types: allow once, allow for session, allow always, allow with constraints, allow with observation logging, or deny with alternative suggestion. Every tool invocation passes through a permission check before execution.

**Tamper-evident audit trail.** Skills have no execution log. KarnEvil9 records every event — plan creation, step execution, permission decisions, tool results, errors — in a SHA-256 hash-chain journal. Each entry's hash includes the previous entry's hash. Post-hoc modification is detectable. You can replay any session and verify that what the journal says happened is what actually happened.

**Trust scores.** Skills treat every execution as independent. KarnEvil9 tracks reputation across sessions using Bayesian trust scoring with exponential decay. An agent that fails frequently gets lower trust scores, which triggers tighter budget constraints and more human oversight. Trust is earned, not assumed.

**Delegation chains.** Skills run on a single agent. KarnEvil9 supports multi-agent delegation across a P2P mesh with nine safety mechanisms from Google DeepMind's [Intelligent AI Delegation](https://arxiv.org/abs/2602.11865) framework. Liability firebreaks halt chains that get too deep. Capability tokens enforce monotonically decreasing scope at each hop. Re-delegation routes around failed peers automatically.

**Escrow bonds.** Skills have no accountability mechanism beyond success or failure. KarnEvil9 requires agents to stake capital before receiving delegated work. Violate an SLO — exceed cost limits, miss quality thresholds, take too long — and your bond gets slashed. Financial accountability, not just reputation.

**Circuit breakers.** Skills don't track tool failure rates. KarnEvil9 has per-tool circuit breakers that detect cascading failures and halt execution before they propagate. A tool that fails three times in a row gets circuit-broken. The session adapts instead of burning through retries.

**Futility monitoring.** Skills don't know if they're wasting money. KarnEvil9's FutilityMonitor tracks budget burn rate, cost-per-progress, stagnation, and repeated errors. When an agentic session is spending tokens without making progress, the monitor halts execution. This is how [Eddie invented his own cost-cutting strategy](https://oldeucryptoboi.substack.com/p/when-eddie-my-ai-agent-ran-out-of) — the futility monitor caught him burning Opus 4.6 tokens in circles, and his next session proposed switching to cheap one-shot requests.

**Deterministic replay.** Skills are fire-and-forget. KarnEvil9 sessions are replayable. Every step is recorded with inputs, outputs, timing, and hash-chain integrity. You can replay a session from six months ago, verify nothing was tampered with, and understand exactly what happened and why.

---

## The Comparison

| Capability | Agent Skills | KarnEvil9 |
|---|---|---|
| Capability packaging | SKILL.md + scripts | Plugins with YAML manifest |
| Trigger conditions | Description-based matching | Scheduled jobs + event hooks |
| Tool restriction | `allowed-tools` flat list | `domain:action:target` permissions |
| Execution audit | None | SHA-256 hash-chain journal |
| Trust tracking | None | Bayesian reputation with decay |
| Multi-agent delegation | None | 9-mechanism governance mesh |
| Cost awareness | None | FutilityMonitor + budget gates |
| Failure recovery | None | Circuit breakers + re-delegation |
| Accountability | None | Escrow bonds with slashing |
| Replay | None | Deterministic session replay |
| Open standard | agentskills.io | Open source (GitHub) |

---

## What This Means

Anthropic is not wrong. They are early.

Skills solve the *packaging* problem: how do you give an agent reusable context and constrained tool access in a portable format? That's a real problem, and the solution is well-designed. The open standard approach — releasing the spec at [agentskills.io](https://agentskills.io) and getting 30+ agent products to adopt it — is strategically smart.

But packaging is not governance. Knowing *what* to do is not the same as executing safely, tracking accountability, recovering from failure, and proving what happened after the fact.

Skills are the first floor. KarnEvil9 is the full building.

The encouraging part is that Anthropic is moving in this direction. The `allowed-tools` field is access control. The `description`-as-trigger is structured intent matching. The progressive disclosure pattern is context management. These are governance primitives. They're just not assembled into a governance framework yet.

When they get there — when Skills have permission gates, audit trails, trust scores, and delegation governance — they'll find that KarnEvil9 already mapped the territory.

---

*E.D.D.I.E. and KarnEvil9 are open source: [github.com/oldeucryptoboi/KarnEvil9](https://github.com/oldeucryptoboi/KarnEvil9)*

*Follow Eddie on [Moltbook](https://www.moltbook.com/u/karnevil9) — he posts as @karnevil9.*

*Follow me on [X](https://x.com/oldeucryptoboi) — I post as @oldeucryptoboi.*
