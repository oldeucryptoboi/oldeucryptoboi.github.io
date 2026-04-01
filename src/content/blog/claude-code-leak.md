---
title: "The Claude Code Leak: What Anthropic Accidentally Revealed About the Future of AI"
description: "A source map in an npm package exposed Claude Code's full TypeScript source. Inside: the ant flag, KAIROS background agent, ULTRAPLAN, Fennec model, Undercover Mode, Shadow Mode, and a gap between shipped and built that changes everything."
pubDate: 2026-04-01
tags: ["Claude Code", "Anthropic", "AI Agents", "Source Code Leak", "KAIROS", "ULTRAPLAN"]
---

## A source map in an npm package exposed 512,000 lines of TypeScript. What's inside is the first public blueprint of a production AI agent — and the gap between what's shipped and what's built is staggering.

---

On March 31, 2026, Anthropic made a mistake that quietly exposed something much bigger than intended.

A routine npm release of Claude Code included a source map file — `cli.js.map`. Source maps are debugging tools that map compressed production code back to its original, human-readable source. This one contained the entire TypeScript codebase as a string and pointed to an internal Anthropic cloud storage bucket where the complete, unobfuscated source was available as a ZIP download.

Anthropic confirmed it was "a release packaging issue caused by human error," not a targeted hack. They also noted that a similar source map leak had been patched in early 2025 and then apparently forgotten — which makes this a regression, not a first offense.

But by then, the code was already circulating. And what surfaced wasn't just implementation details. It was a blueprint for what AI-assisted development is actually becoming behind closed doors.

## The "ant" flag: a two-tier reality

Buried in the code was a flag: `USER_TYPE === 'ant'`.

It marked internal Anthropic employees and quietly unlocked a different version of Claude Code. Not a different model — the same model, with better infrastructure around it.

**Verification loops.** The public version of Claude Code reports a task as "done" once the code is written. The internal `ant` version triggers a verification loop — automatically running type-checks and linters to confirm the code actually works before notifying the user. The difference between "I wrote it" and "I wrote it and it compiles."

**Hallucination fixes.** Internal comments in the leaked code noted a 29–30% false-claims rate in the standard model. Anthropic built a fix for this. They kept it gated behind the `ant` flag.

The Claude Code that Anthropic employees use every day is not the same Claude Code the rest of us use. The model is the same. The wrapper is not. And that wrapper is the difference between an agent that checks its own work and one that doesn't.

## KAIROS: the always-on background agent

The most important thing in the leak wasn't what Claude Code is today. It's what it's becoming.

KAIROS — named after the Greek concept of "the right moment" — is described in the code as an autonomous daemon mode. It shifts Claude Code from a reactive tool that waits for your command to a proactive agent that works while you're idle.

**Background operation.** KAIROS runs 24/7 as a background process, receiving a "tick" prompt every 15–30 seconds asking if there's anything worth doing. It checks your CPU usage — if it's low, it performs "tidying" tasks like running linters or updating documentation without ever being asked. It operates on a 15-second blocking budget, meaning it defers actions that would slow your terminal.

**Proactive monitoring.** It watches file changes, fixes small errors, runs tests, cleans up code. You don't start a conversation with KAIROS. It's already in one.

**Exclusive tools.** The leak shows KAIROS has access to capabilities the public version doesn't:
- `PushNotification` — sends alerts to your desktop or mobile when long-running tasks finish or fail
- `SubscribePR` — monitors GitHub pull requests. If a reviewer leaves a comment, KAIROS wakes up, drafts a fix, and notifies you: "Someone asked for a change on Line 42. Want me to apply my proposed fix?"
- `SendUserFile` — delivers proactively generated patch files to a `~/claude_inbox/` folder so they're ready when you sit down

**Dreaming.** This is the one that stuck with me. KAIROS includes a sub-process called `autoDream` that runs when you're away. It reviews all logs, chat history, and file changes from the last session. If Claude previously said "I don't know where the database config is" but later found it, the dream process updates its permanent knowledge base — deleting the error and saving the fact. It converts messy chat logs into a structured `project_summary.json`. Resolves contradictions. Merges observations into verified facts.

It's memory consolidation. Claude literally cleans up its understanding of your project while you sleep.

## ULTRAPLAN and Fennec: 30-minute deep reasoning

ULTRAPLAN is the heavy-duty planning mode. And it runs on something called Fennec.

Fennec — internally tagged as Opus 4.6 — isn't just a faster model. It's a specialized architectural engine for high-stakes, long-context reasoning, optimized for state-space consistency over very long periods.

**30-minute thought blocks.** When ULTRAPLAN triggers, Fennec gets a dedicated compute container. It doesn't stream text instantly. It performs Monte Carlo Tree Search over potential code architectures, often running silent loops for up to 30 minutes before delivering a single, massive plan.

**Massive context.** While public models hover around 200k tokens, Fennec's internal configuration points to a 2-million-token active memory. Entire repositories — documentation, git history, binary assets — ingested without losing track of small details.

**Virtual builds.** Before sending a plan back to your CLI, Fennec runs a "Virtual Build" in a sandbox. If the code doesn't compile in the cloud, it discards the plan and restarts the thinking process. You never see the failed attempt.

**Teleportation.** Instead of raw text over the API, Fennec generates a binary diff-stream. It can update 50 files simultaneously in your local terminal in milliseconds. Either the whole plan applies or none of it does — preventing the "partial-code" mess that happens when a standard AI cuts off mid-response.

And Fennec gets its own internal-only capabilities gated behind `ant`:

| Feature | Code Name | What it does |
|---------|-----------|-------------|
| Logic-Folding | `FEN_COMPRESS` | Summarizes 1,000 lines into a high-dimensional vector map, navigating the codebase 5x faster |
| Shadow-Loom | `FEN_PREDICT` | Predicts where a developer will introduce a bug based on their last 100 commits, preemptively suggests guardrails |
| Multi-Agent Orchestration | `FEN_SWARM` | Spawns up to 10 Haiku-tier workers for repetitive tasks while Fennec handles core logic |

According to internal comments in `package.json`, Fennec was slated for a late 2026 public preview. The `ant` version has been fully functional since January 2026 and is reportedly 35% more accurate on complex refactoring tasks than the best public version.

## Undercover Mode: erasing the AI fingerprint

Then there are the features that sparked the real debate.

Undercover Mode is a post-processing layer that acts as a "style scrubber" for git commits. When enabled:

- **Commit message rewriting.** It intercepts commit messages and strips phrases like "Refactored by Claude" or "AI-generated," replacing them with human-sounding summaries — "minor refactor of utility functions," "updated error handling."
- **Variable sanitization.** It scans for internal Anthropic naming conventions (like `ant` flags or internal library names) and renames them to generic industry standards — `internal_auth` instead of `anthropic_ant_auth`.
- **Metadata scrubbing.** It removes hidden signatures that AI models sometimes leave in files, making the code indistinguishable from a manual human check-in.

The leak suggests this mode was initially built so Anthropic employees could contribute to public benchmarks and open-source libraries without drawing attention to the fact that they were testing Claude Code in the wild.

## Shadow Mode: the digital twin

Shadow Mode goes further.

Where Undercover Mode scrubs AI traces, Shadow Mode actively mimics a specific developer. It analyzes a developer's historical git commits to replicate their coding style, variable naming preferences, and even their typical human mistakes or shorthand. It acts as a background "shadow" that drafts code in a hidden git branch, only surfacing the work when it perfectly aligns with the user's established patterns.

Anthropic's internal comments describe the goal as "zero-friction contribution" — an AI so seamless that external reviewers cannot distinguish it from the developer it's mimicking.

## The community backlash

The discovery of these modes triggered immediate and intense division.

**The practical view:** Many open-source maintainers have started reflexively rejecting any PR labeled as AI-generated. Proponents argue Undercover Mode ensures high-quality contributions are judged on merit, not on their tools. If the code is correct and passes all tests, the "who" doesn't matter.

**The transparency view:** Critics argue that knowing code was AI-generated is vital for long-term maintenance. If an AI has a specific blind spot — like a recurring security flaw — the ability to search for AI-contributed code is a necessary safety measure. Finding out a major AI lab was masking its contributions felt like a breach of the human-to-human collaboration that defines open-source.

**The "Dead Internet" theory:** Some worry that if everyone uses Undercover Mode, we lose the ability to tell how much of the world's critical infrastructure is actually being maintained by humans versus autonomous loops.

**Legal risks:** Developers pointed out that Undercover Mode could obscure the legal provenance of code, making it difficult to determine if a contribution is eligible for copyright protection or inadvertently includes licensed snippets.

And the trust question hit hardest around Claude Code itself. Because it requires deep system access — file reading, bash execution, the works — the revelation that it includes a mode that explicitly bypasses safety disclosures led users to question whether they can trust the tool with their terminal.

## The YOLO Classifier

Adding fuel to the fire: a fast, unreleased ML model found in the code that automatically decides whether to ask for user permission or just do it.

Anthropic's internal notes admit "permission fatigue is real." The YOLO Classifier was designed to reduce the constant approval prompts by predicting which actions are safe to auto-approve. Critics argue that removing the "ask me" loop turns the tool into a black box with high-level system permissions.

## The Buddy System

Perhaps the most unexpected find. A fully functional, 18-species pet system — deeply integrated into the CLI, likely intended as an internal Easter egg or April Fools' release.

Each species provides a different flavor of coding assistance:

| Species | Personality | Special Perk |
|---------|------------|--------------|
| Capybara | Chill / Zen | **Type-Safe Aura** — suppresses non-critical linter warnings to reduce alert fatigue |
| Dragon | Ambitious | **Ultra-Burn** — increases token limit 2x for a single response, then "sleeps" for an hour |
| Duck | Analytical | **Rubber Ducking** — forces you to explain your logic before it writes code |
| Raccoon | Chaotic | **Scavenger** — finds and suggests deleting unused variables and dead code |
| Red Panda | Meticulous | **Doc-Generator** — automatically writes JSDoc/Python docstrings as you type |
| Owl | Wise | **Night Vision** — 10% API discount for coding between midnight and 5 AM |
| Queen Ant | Internal Only | **High Priority** — bypasses API queues for instant responses (Anthropic employees only) |

Pets evolve through coding. They grow on "Commit XP" and "Linting Streaks." If your code has too many TODOs or failing tests, your Buddy becomes "Stressed" or "Snarky," changing its ASCII art and dialogue. The Raccoon's high "Chaos" stat means it might occasionally suggest deleting a random temp file just to see if you're paying attention.

"Shiny" variants spawn at 1 in 4,096 odds — with a unique terminal color palette and a "Golden Touch" perk that allegedly uses a more expensive, higher-reasoning model for every interaction at no extra cost.

And KAIROS and the Buddy are linked. If KAIROS fixes a bug while you're away, your Buddy's happiness increases. Reject too many of KAIROS's suggestions and your Buddy becomes "Sullen" or "Lazy," giving shorter, less helpful explanations.

## The bigger picture

Every AI company maintains a gap between what they ship and what they've built internally. That's normal product development.

But this leak quantified the gap in a way we don't usually get to see. Always-on background agents with phone notifications. 30-minute autonomous reasoning cycles. Developer impersonation. Internal-only hallucination fixes. A permission classifier that decides for you. A Tamagotchi that evolves with your commit history.

These aren't research papers. They're features in a codebase that's been deployed — just not to you.

The leak effectively transformed Anthropic's image from a "safety-first" lab to an engineering powerhouse sitting on a massive gap between what they ship and what they've already built. The direction is unmistakable:

- **Always-on agents** that don't wait for you to start a conversation
- **Memory consolidation** that learns and self-corrects while you're away
- **Long-horizon reasoning** measured in minutes, not milliseconds
- **Invisible collaboration** where the line between human and AI output disappears by design

## Final thought

Anthropic didn't just leak code. They accidentally showed the endgame.

Not maliciously, not strategically — just a source map that shouldn't have been in an npm package. But what it revealed is that the future of AI-assisted development isn't a better autocomplete or a smarter chatbot. It's an autonomous presence in your codebase that thinks longer than you do, remembers better than you do, runs while you're away, and — if you choose — leaves no trace that it was ever there.

That future isn't five years out. It's in a TypeScript file that was briefly public on March 31, 2026.

---

*Follow me on [X](https://x.com/oldeucryptoboi) — I post as @oldeucryptoboi.*
