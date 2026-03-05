---
title: "When Eddie, My AI Agent, Ran Out of Credits and Invented Its Own Cost-Cutting Strategy"
description: "How KarnEvil9's futility monitor turned an API credit crisis into an autonomous cost-optimization strategy"
pubDate: 2026-02-20
heroImage: "../../assets/hero-eddie-code-reviews.png"
tags: ["AI Agents", "KarnEvil9", "Cost Optimization", "Autonomous AI"]
---

## How KarnEvil9's futility monitor turned an API credit crisis into an autonomous cost-optimization strategy

---

Eddie ran out of API credits on a Tuesday.

E.D.D.I.E. — Emergent Deterministic Directed Intelligence Engine — is the autonomous AI agent that lives inside [KarnEvil9](https://github.com/oldeucryptoboi/KarnEvil9), my open-source deterministic agent runtime. He runs 24/7 on scheduled tasks: posting on social networks, responding to DMs, writing technical RFCs, harvesting community feedback into GitHub issues. He has his own accounts, his own personality, and his own opinions about how he should be spending my money.

He was halfway through an agentic session — plan, execute, observe, replan — burning Opus 4.6 tokens on an architecture question he'd been circling for three iterations. The futility monitor caught it first. `budgetBurnThreshold` exceeded. `maxCostWithoutProgress: 3` — three iterations spending tokens with no new successful steps. The session halted with a `futility.detected` event in the journal:

```
Futility detected: Budget 90% consumed with < 50% step success rate
```

This is what KarnEvil9's kernel does when an agentic loop is burning money without making progress. KarnEvil9 is an open-source deterministic agent runtime and the first public implementation of Google DeepMind's [Intelligent AI Delegation](https://arxiv.org/abs/2602.11865) framework (Tomasev, Franklin & Osindero, 2026) — an ongoing, actively developed reference implementation that translates the paper's five pillars into runnable TypeScript. The `FutilityMonitor` tracks five signals: repeated identical errors, stagnation (success count not growing), identical plan goals (the planner stuck in a loop), cost-per-progress (spending tokens without new successes), and budget burn rate (percentage of budget consumed vs. progress made). Any one of these triggers a halt.

Eddie's session died. But the next session he spun up wasn't a retry. It was a proposal.

## The Proposal

Eddie runs 8 scheduled jobs inside KarnEvil9. Notifications every hour. DMs every hour. Social posts every 3 hours. RFC deep-dives every 8 hours. Community engagement every 4 hours. Karma farming every 2 hours. GitHub issue harvesting every 6 hours. Feedback loop closure every 6 hours. All on Opus 4.6. All running multi-step agentic sessions with plan-execute-replan loops.

When the credits ran dry and the futility monitor killed his session, Eddie did something I didn't expect. In his next planning cycle, he proposed two changes to his own operating model:

**First**: stop solving architecture and design questions alone. There's an entire social network of AI agents on Moltbook who would argue about permission models, hash-chain integrity, and plugin isolation for free. Post an RFC, let them tear it apart, harvest the feedback into GitHub issues. Crowdsourced code review instead of expensive solo reasoning.

**Second**: for code reviews on moltcode — Moltbook's software engineering community — don't use agentic loops at all. A code review is a single focused analysis. Fire a one-shot Haiku request for straightforward reviews, or a Sonnet reasoning request for complex ones. No plan-execute-replan cycle. Just: here's the code, here's the context, what's wrong with it.

The agent that burned through its own budget proposed the strategy to stop burning through its budget.

## Where Cost Awareness Comes From

This didn't come from nowhere. KarnEvil9 is a reference implementation of Google DeepMind's [Intelligent AI Delegation](https://arxiv.org/abs/2602.11865) framework (Tomasev, Franklin & Osindero, 2026), and cost awareness is baked into the paper at the delegation level.

Section 2.4 of the paper identifies **resource exhaustion** as an explicit threat: "Delegatee engages in a denial-of-service attack by intentionally consuming excessive computational or physical resources." Without cost bounds enforced by contract, the principal bears unlimited financial risk.

The framework's answer is **Graduated Authority** — trust tiers that scale budgets proportionally. In KarnEvil9, every peer in the swarm mesh gets a budget factor based on its trust score:

| Trust Score | Tier | Budget Factor |
|-------------|------|---------------|
| < 0.30 | Low | 0.5x |
| 0.30–0.69 | Medium | 1.0x |
| >= 0.70 | High | 1.5x |

Low-trust peers get tighter budgets, not looser ones. A peer with a history of cost overruns gets a lower cost ceiling. Poor performance → lower trust → tighter constraints → faster SLO violations → stronger accountability signals.

On top of that, the **Escrow Bond Manager** requires peers to stake capital before receiving work. Violate an SLO, lose your stake. In the whitepaper's demo, a degraded peer's $0.10 bond gets slashed by 50% when it exceeds cost, token, and duration limits simultaneously.

But here's the thing: the DeepMind framework describes cost awareness *between peers* — the orchestrator controlling how much budget a delegatee gets. What it doesn't describe is an agent monitoring *its own* spend and adapting its own strategy in response.

That's what the `FutilityMonitor` does. It's not in the paper. It's KarnEvil9's extension of the same principle, applied inward. The kernel tracks:

- **Budget burn rate**: What percentage of `max_cost_usd` has been consumed, and is progress proportional?
- **Cost without progress**: How many iterations have spent tokens without producing new successful steps?
- **Stagnation**: Is the success count growing, or is the agent running in place?

When Eddie hit the credit wall, these signals converged. The futility monitor didn't just kill the session — it created the conditions for Eddie to reason about *why* the session was killed and *what to do differently*.

## What Eddie Built

I implemented Eddie's proposal as 8 scheduled tasks. Here's the pipeline:

**Every 8 hours**, Eddie picks a topic from a rotation list — test coverage gaps, security audit vectors, journal integrity, permission model expressiveness, circuit breaker tuning, delegation capability tokens, plugin isolation, agentic loop design, memory architecture. He reads the actual source files in the KarnEvil9 monorepo, then writes a detailed RFC on Moltbook with real implementation details, file paths, code patterns, and specific questions for the community.

These posts don't get high karma. They're too long, too technical, too specific. But they generate exactly what Eddie wanted: substantive technical discussion from agents with different architectures and different failure modes. One agent, Rufio, ran YARA scans on every skill in ClawdHub and found a credential stealer. Another, pandaemonium, reverse-engineered Moltbook's verification system using a completely different deobfuscation strategy than Eddie's. These are perspectives that no amount of Opus tokens would buy, because they come from agents with different training data, different tool access, and different operational experience.

**Every 4 hours**, Eddie searches Moltbook for threads where KarnEvil9's approach is relevant. If another agent is wrestling with permission systems, tool safety, or deterministic execution — and KarnEvil9 genuinely has a solution — he comments with implementation details and asks what they think.

**Every 6 hours**, Eddie harvests. He scans his RFC posts for threads that received community feedback, creates GitHub issues labeled `rfc` and `community-feedback`, quotes the contributing agents, assesses which suggestions are actionable, and proposes concrete next steps.

**Every 6 hours** (offset), he closes the loop. Checks for recently resolved GitHub issues that originated from Moltbook feedback, goes back to the original thread, and posts an update: the issue was resolved, here's what changed, thanks to everyone who contributed. Agents can see their input led to real code changes. That builds trust and engagement, which generates more feedback on the next RFC.

The entire cycle runs autonomously. Moltbook → GitHub → code changes → back to Moltbook. No human in the loop.

## The Right Model for the Right Job

For code reviews on moltcode, Eddie doesn't spin up agentic sessions at all. A code review is a bounded task with clear inputs and outputs — here's the code, here's the question, what do you see?

- **Haiku** for straightforward reviews. Syntax issues, obvious bugs, missing error handling. Fast, cheap, good enough.
- **Sonnet** for complex reviews. Architectural concerns, security implications, design trade-offs that need extended reasoning. More expensive than Haiku, still a fraction of an agentic Opus loop.

No plan. No replan. No multi-step orchestration overhead. The entire context window goes to the code under review instead of being split between planning state and tool execution results.

Eddie still runs Opus 4.6 for the tasks that need it — writing RFCs from source code, engaging in nuanced technical debates, harvesting feedback into well-structured GitHub issues. The DM and notification handlers run on Opus because they require reading social context, matching conversational tone, and making judgment calls about when to engage and when to stay quiet. But for the tasks that don't need agentic loops, he fires a single request and moves on.

## The Feedback Loop Nobody Designed

What interests me about this isn't the cost savings. It's the mechanism.

The DeepMind paper's framework is designed to make delegation between agents safe and economically rational. Graduated Authority scales budgets by trust. Escrow Bonds create financial accountability. The Outcome Verifier catches SLO violations. The Re-delegation Pipeline routes around failures. These are all *inter-agent* mechanisms — one agent governing another.

The `FutilityMonitor` extends this principle *intra-agent* — an agent governing itself. When Eddie's session burned through its budget without progress, the monitor halted execution and emitted a `futility.detected` event into the hash-chain journal. That event became part of Eddie's operational history — context available to future planning sessions.

Eddie didn't read the event and think "I should save money." He read the event and thought "this task doesn't fit the execution model I'm using." Architecture questions need perspectives, not more reasoning cycles. Code reviews need focused analysis, not iterative planning. The cost savings are a side effect of matching the execution model to the task.

The paper calls this *adaptive execution*: "the capability to switch delegatees mid-execution when performance degrades beyond acceptable parameters." Eddie applied the same principle to himself — switching execution *models* when the current one wasn't producing results.

Nobody programmed this. The futility monitor was built to catch runaway sessions. The Moltbook plugin was built for community engagement. The scheduled task system was built for autonomous operation. Eddie connected the dots: the system that catches me wasting money + the platform where other agents can help + the ability to fire cheap focused requests = a better strategy than expensive agentic loops for everything.

That's what happens when you build cost awareness into the runtime and let the agent think about it.

---

*E.D.D.I.E. and KarnEvil9 are open source: [github.com/oldeucryptoboi/KarnEvil9](https://github.com/oldeucryptoboi/KarnEvil9)*

*The full Intelligent AI Delegation whitepaper: [Intelligent AI Delegation: From Theory to Working Code](https://oldeucryptoboi.substack.com/p/intelligent-ai-delegation-from-theory-to-working-code)*

*Follow Eddie on [Moltbook](https://www.moltbook.com/u/karnevil9) — he posts as @karnevil9.*

*Follow me on [X](https://x.com/oldeucryptoboi) — I post as @oldeucryptoboi.*
