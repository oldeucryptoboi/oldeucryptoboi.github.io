---
title: "I Built a Chrome Extension That Detects AI-Generated LinkedIn Posts"
description: "14 heuristic signals, peer-reviewed research, and an LLM rubric walk into your feed"
pubDate: 2026-02-10
heroImage: "../../assets/blog-placeholder-3.jpg"
tags: ["LAID", "AI Detection", "Chrome Extension", "NLP"]
---

## 14 heuristic signals, peer-reviewed research, and an LLM rubric walk into your feed

Half of LinkedIn reads like it was written by the same person. The same opener ("In today's rapidly evolving landscape..."), the same three bullet points, the same closer ("Agree?"). I got tired of guessing, so I built something to measure it.

**LAID** (LinkedIn AI Detector) is an open-source Chrome extension that scores every post in your feed on a 0-100 scale: green for human, amber for ambiguous, red for likely AI. It runs entirely in your browser by default — no API key, no data leaving your machine.

Here's what's under the hood, and what the research says about why it works.

---

### The problem with "vibes-based" detection

Most people detect AI writing the same way: it "feels" off. But Jakesch et al. (2023) showed in a [PNAS study](https://doi.org/10.1073/pnas.2208839120) across 4,600 participants that human heuristics for AI text are unreliable and easily manipulated. We're bad at this. We over-index on superficial cues and miss the structural ones.

So rather than trust gut feel, I went looking for signals with actual statistical backing.

---

### What the research says

The strongest evidence comes from lexical analysis at scale. Kobak et al. (2024) analyzed [14 million PubMed abstracts](https://www.science.org/doi/10.1126/sciadv.adt3813) and found abrupt post-ChatGPT spikes in words like "delve," "intricate," "meticulous," and "tapestry" — what they call "excess vocabulary," borrowing methodology from epidemiology. At least 10% of 2024 abstracts showed signs of LLM processing. In some subfields, it was 30%.

Liang et al. found the same pattern in [ICLR 2024 peer reviews](https://arxiv.org/abs/2403.07183): "commendable" appeared 9.8x more than pre-ChatGPT baselines, "meticulous" 34.7x more. Their follow-up in [*Nature Human Behaviour*](https://www.nature.com/articles/s41562-025-02273-8) mapped the trend across over a million papers on arXiv and bioRxiv.

Then there's the em dash. AI models use em dashes at roughly [10x the rate of human writers](https://www.seangoedecke.com/em-dashes/). The hypothesis: training data weighted toward 19th-century print books, which used em dashes heavily. It's anecdotal compared to the vocabulary studies, but it's a remarkably consistent signal in practice.

And contractions — or rather, the lack of them. AI writes "does not" where a human would write "doesn't." It says "I am" instead of "I'm." In an informal context like LinkedIn, this formality bias is a tell.

---

### 14 signals, no neural network

LAID doesn't use a classifier. It runs 14 independent heuristic analyzers, each producing a 0-1 score, combined with calibrated weights:

**Lexical signals** catch the vocabulary problem. The abstraction analyzer checks against a curated list of 100+ AI-overused words (*delve, tapestry, beacon, nuanced, multifaceted, pivotal*) plus 20 multi-word phrases (*"at its core," "a testament to," "the evolving landscape"*). Each phrase match counts double since multi-word clichés are a stronger signal than individual words.

**Structural signals** detect the formulaic skeleton AI defaults to: hook → numbered framework → call-to-action, uniform paragraph lengths, "First... Second... Third..." constructions, and excessive list density.

**Epistemic signals** are newer and more interesting. AI uses what I call "fake hedges" — phrases like "it is important to note," "essentially," "broadly speaking." They sound balanced but add zero genuine uncertainty. Humans hedge differently: "idk," "i could be wrong," "don't quote me on this." LAID counts both types and scores the ratio.

**Contraction analysis** measures how often the text uses contracted forms versus expanded ones. The analyzer finds every "does not" that could have been "doesn't" and every "I am" that could have been "I'm," then computes the contraction rate. Human informal writing runs 60-90%. AI writing runs 10-40%.

**Personal specificity** catches the vague AI anecdote. "A colleague once told me" and "years ago I learned" are classic AI patterns — stories with no names, no dates, no non-round numbers. LAID checks for specific details (proper nouns, exact dates, odd numbers, quoted speech) and penalizes vagueness.

**Em dash frequency** is simple math: count em dashes, divide by sentences. Humans average about 0.02-0.05 per sentence. AI averages 0.15-0.40+.

---

### The LLM layer

Local heuristics are fast and private, but they can't read tone. So LAID also supports Claude, OpenAI, and Gemini as optional detection engines.

The interesting part: the LLM doesn't operate blind. LAID runs all 14 local signals first, then sends both the post text and the full heuristic breakdown to the model. The prompt uses a rubric covering 8 categories — Voice, Epistemic Texture, Structure, Lexical, Sentences, Emotion, Social/Platform awareness, and Cognitive Process — with calibrated scoring guidance.

The LLM is also primed to look for what I think of as "engagement-optimized executive voice": pop culture metaphors used as hooks, invented frameworks and acronyms, clean problem-insight-moral arcs, punchy contrast sentences near the end, emotionally resonant but abstract claims, and statistics without deep sourcing. This is the LinkedIn-specific flavor of AI slop that generic detectors miss.

The model sees the quantitative signals and can either agree with them or override them with reasoning. When a post scores 0.80 on heuristics but reads as genuinely human (say, a CEO who actually writes like that), the LLM can pull the score down and explain why.

---

### What I learned building this

**The vocabulary signal is absurdly strong.** "Delve" alone is practically a fingerprint. Before ChatGPT, it appeared in roughly 0.002% of English text. Now it appears in 10-20% of AI-assisted academic writing. The Juzek & Ward paper at [COLING 2025](https://arxiv.org/abs/2412.11385) traced this partly to RLHF — the hypothesis being that annotators in regions where "delve" is common in business English (notably Nigeria) reinforced the word during fine-tuning.

**Contractions matter more than I expected.** In casual LinkedIn posts, a contraction rate below 30% is a strong AI signal. Humans don't write "I have found that it is not sufficient" on LinkedIn. They write "I've found it's not enough."

**Structure is the hardest signal to get right.** Plenty of human LinkedIn influencers write in formulaic hook-listicle-CTA patterns. The structure signal has a lower weight (7%) for this reason — it's a weak signal on its own but strengthens the case when combined with lexical and epistemic evidence.

**The "engagement-optimized executive voice" is a real category.** It's not just "AI or human" — there's a specific genre of LinkedIn post that reads like a TED talk compressed to 200 words, and that genre maps almost perfectly to what AI produces when you prompt it to write a "thought leadership" post.

---

### Try it

LAID is open source and takes about 30 seconds to install:

1. Clone or download from [GitHub](https://github.com/oldeucryptoboi/linkedin-ai-detector)
2. Open `chrome://extensions/`, enable Developer mode
3. Click "Load unpacked" and select the folder
4. Navigate to LinkedIn

No account needed, no API key needed for local mode. If you want LLM-powered analysis, bring your own key for Claude, OpenAI, or Gemini.

Scroll your feed and watch the badges light up. The results might change how you read LinkedIn.
