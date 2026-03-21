---
title: "Claude Agent SDK v1 vs v2: Control vs Delegation"
description: "The older SDK feels like building a machine. The v2 preview feels like delegating work. Same tools, very different philosophy."
pubDate: 2026-03-21
tags: ["AI Agents", "Claude", "Anthropic", "SDK"]
---

<!-- seo_title: Claude Agent SDK v1 vs v2: Control vs Delegation -->
<!-- seo_description: The older SDK feels like building a machine. The v2 preview feels like delegating work. Same tools, very different philosophy. -->
<!-- slug: claude-agent-sdk-v1-vs-v2 -->

## The older SDK feels like building a machine. The v2 preview feels like delegating work. Same tools, very different philosophy.

The [Claude Agent SDK](https://docs.anthropic.com/en/docs/agents-and-tools/claude-agent-sdk) and the new [v2 preview](https://docs.anthropic.com/en/docs/agents-and-tools/agent-sdk-v2) look similar on the surface, but they don't behave the same at all.

---

### Wired by hand

The older SDK feels like building a machine. Every step is wired in. You decide when a tool gets called, what happens after, how errors are handled, when the whole thing stops. Nothing moves unless you tell it to.

That level of control is nice… until it isn't.

After a couple of days, the code starts to feel repetitive. Same branching patterns, same retry logic, same guardrails copied across projects. It works, it's predictable, but it's also kind of exhausting to maintain.

---

### Delegated by design

The v2 preview shifts that responsibility.

Instead of scripting every move, you give it a goal, access to tools, and some boundaries. Then it runs. It can decide to call multiple tools in sequence, backtrack if something fails, even try a different approach — without being explicitly told to.

It feels lighter. Also less certain.

There's a moment the first time it "just handles something" on its own where you pause a bit. Part of you is impressed. Part of you is wondering what else it's doing that you didn't plan for.

---

### The tradeoff

That's really the tradeoff.

The older SDK is closer to [deterministic software](https://en.wikipedia.org/wiki/Deterministic_system). You know the path because you wrote the path.

The v2 preview is closer to delegated work. You describe the outcome, not every step along the way. Most of the time it gets there. Sometimes it takes a route you wouldn't have chosen.

And yeah, sometimes it gets weird.

---

### When to use which

For anything strict — like workflows where every step matters — the older approach still feels safer. There's comfort in knowing nothing unexpected will happen.

For open-ended tasks, the kind where you'd normally have to write a bunch of branching logic just to cover edge cases, the [agentic model](https://arxiv.org/abs/2402.01680) starts to make more sense.

It's not really about better or worse.

It's about whether you want to control the process… or hand over the process and just define the boundaries.
