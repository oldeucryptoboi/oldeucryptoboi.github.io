---
title: "Coding with AI Feels Like Leaf Blowing"
description: "AI tools work the way leaf blowers work. The piles are real. The leaves keep falling."
pubDate: 2026-05-09
heroImage: "../../assets/hero-leaf-blowing.png"
heroAlt: "Comic-book panel labeled 'The Entropy Storm!' showing a caped engineer firing a GENERATE-branded leaf blower into a vortex of code files, prompts, error popups, and folder icons."
tags: ["AI", "Software Engineering", "Claude Code", "Productivity"]
---

## AI tools work the way leaf blowers work. The piles are real. The leaves keep falling.

Coding with AI tools is starting to feel like leaf blowing. Not the metaphor, the actual thing. I have a Stihl BG 50 in the garage. Last October I spent maybe three hours blowing leaves off a quarter-acre lot. The pile by the curb was enormous. Two days later you couldn't tell I'd done it. That's the part I keep coming back to.

Leaf blowers work. They're loud, fast, they move material. The piles are real. But the leaves keep falling, and a pile is not a finished yard. You did something. You moved something. You will do it again next weekend.

Three weeks ago I asked Claude Code to add one subcommand to a CLI I'd been writing. Maybe forty lines of work if I'd done it by hand. What I got was a new module, three new test files, an unsolicited refactor of the option parser, and two helper functions that already existed under different names elsewhere in the repo. It all ran. The tests passed. I closed the laptop feeling like something had happened. On Monday I had to read the diff with someone else and I couldn't explain half of it.

The actual typing was never the bottleneck. I think most of us knew that already, but it's a different thing to feel it. The bottleneck was always whether the people who own this code understand it well enough to change it next quarter without breaking something. AI tools didn't solve that. They made the other side cheaper, and now the ratio is off in a way it wasn't before.

So you end up doing custodial work. Pruning. Re-reading. Rejecting. Asking the model to do less. A lot of the engineers I respect are dialing scope down right now, not up. Smaller PRs. Tighter specs. More boilerplate they used to delegate, written by hand, because writing it by hand is how they keep the model of the system in their head.

People call this orchestration. I don't buy that word. Orchestration suggests a score and a downbeat. This is closer to yard work. You clear one corner, the wind shifts, the leaves are back.

The tools are genuinely productive for migrations, scaffolding, framework boilerplate, the kind of work where you'd normally read documentation for two hours before writing anything. That gain is real and I don't think it goes away. The strange part is the tiredness. You can spend a whole afternoon moving in eight different directions, prompts and diffs and re-runs and "actually no, undo that part," and at six p.m. you're tired in a way that doesn't match the size of the change you shipped. Some days the codebase is genuinely better. Some days it's just bigger and you can't tell which corner you displaced the complexity into.

I'm not sure what the right response is. Conservatism feels right, but it also feels like the easy answer, the one a forty-something engineer would obviously give. Maybe the people who grew up on these tools will be fine and I'm just describing what it looks like to learn a new instrument badly.

Either way: the work is cheap, the comprehension is expensive, and right now nobody I know has figured out how to keep the second one from compounding.
