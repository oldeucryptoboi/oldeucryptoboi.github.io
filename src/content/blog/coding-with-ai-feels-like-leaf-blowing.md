---
title: "Coding with AI Feels Like Leaf Blowing"
description: "AI tools work the way leaf blowers work. The piles are real. The leaves keep falling."
pubDate: 2026-05-09
heroImage: "../../assets/hero-leaf-blowing.png"
heroAlt: "Comic-book panel labeled 'The Entropy Storm!' showing a caped engineer firing a GENERATE-branded leaf blower into a vortex of code files, prompts, error popups, and folder icons."
tags: ["AI", "Software Engineering", "Claude Code", "Productivity"]
---

## AI tools work the way leaf blowers work. The piles are real. The leaves keep falling.

I have a leaf blower in the garage. I take it out a few times a year. It's loud, the neighbors don't love it, and the work is satisfying for about an hour after I'm done. By the end of the week the leaves are back. That's not a complaint, it's just how leaves work. Coding with AI tools is starting to feel like that, which is annoying because I don't think the tools are bad. They produce real code. The output exists in a way the input didn't.

Last week I asked Claude Code to add one subcommand to a CLI I keep around for myself. What I got back was the subcommand, a refactor of the option parser I hadn't asked for, and two helper functions that turned out to be near-duplicates of helpers already in the file. It worked. Tests passed. When I tried to walk a colleague through the diff later that afternoon, I lost my place twice. The thing nobody really told us, or maybe everybody did and we ignored it because the typing felt slow at the time, is that typing was never the bottleneck. The hard part was always keeping the system small enough in your head that you could change it later without breaking it. AI doesn't help with that. There's more code now per unit of intent, which means more to hold in your head, not less.

So you end up doing custodial work. Pruning. Re-reading. Rejecting. Asking the model to do less, or sometimes more, when you catch it silently cutting a corner and using a fallback or a mock instead of building the real thing. A lot of the engineers I respect are dialing scope down right now, not up. Smaller PRs. Tighter specs. More boilerplate they used to delegate, written by hand, because writing it by hand is how they keep the model of the system in their head. Underneath all of this is a pace mismatch that's hard to describe. The model produces text faster than I can read with comprehension. So I either skim, in which case mistakes get past me, or I don't skim, in which case I'm reading code at someone else's pace, all day. I read someone describe this as a slot machine effect, which is unkind but accurate. You pull, something happens, you don't fully read it, you pull again.

![A tired man in a torn shirt slumped against a slot machine in a neon-lit casino](slot-machine-gambler.webp)

People call this orchestration. I don't buy the word. Orchestrators have a score in front of them. What I'm doing is closer to yard work, where you blow leaves into one corner and the wind moves them somewhere else and at no point are you really in charge of the wind.

None of which is to say the tools are useless. For migrations, scaffolding, the kind of glue code you'd otherwise have to read AWS docs for two hours to write, the productivity gain is real. I've shipped things in a day that used to take me a week. And yet by 6 p.m. on those days I'm tired in a way that doesn't match what I built. Three hours of what I thought of as work, and what was actually happening was a hundred small judgments about whether to accept a generated suggestion. None of them feel hard alone. I don't have a good name for the cumulative version of that. Decision fatigue isn't quite right, because I'm not really deciding, more like consenting.

I'm not sure what the right reaction is. Smaller scopes, tighter specs, probably, like everyone keeps saying, though I'd be lying if I said I'd actually changed how I work. A few people I respect have moved to local models on big Macs specifically because the slower token rate forces them to read along. That sounds extreme until you notice that the alternative is the slot machine. What we got was cheaper generation without cheaper comprehension, and I don't see how the second one comes down on its own.
