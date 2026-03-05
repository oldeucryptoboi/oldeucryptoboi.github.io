---
title: "Memory-Assisted Learning vs Recursive Language Models: An Empirical Study"
description: "Can an LLM improve at a game by reading its own past failures? A 72-game tic-tac-toe experiment using a swarm-based testbed shows hybrid memory+code beats pure prompt-recursive self-improvement."
pubDate: 2026-02-28
heroImage: "../../assets/hero-memory-vs-rlm.png"
tags: ["AI Agents", "KarnEvil9", "Machine Learning", "Tic-Tac-Toe", "Research"]
---

## Can an LLM improve at a game by reading its own past failures?

**72 games. 72 lessons (36/phase). 6 experimental runs.**

Dependencies: Claude Haiku 4.5, Claude Sonnet 4.5, KarnEvil9 Swarm Package

## Abstract

This report compares two approaches for enabling an LLM to improve at strategic games without weight updates:

**(A) Memory-Assisted Learning (MAL):** Combines LLM reasoning with deterministic code rules derived from accumulated lessons, allowing rule-based execution for specific decision classes.

**(B) Prompt-recursive lessons:** Appends the model's own prior analyses back into its prompt context; all decisions remain language-mediated.

Both systems involve an LLM playing tic-tac-toe as player O (moving first) against a deterministic minimax opponent playing X (moving second) with a 10% random blunder rate.

**Key Results:**

| Metric | MAL | Prompt-recursive |
|--------|-----|------------------|
| Run 1 loss rate | 41.7% | 41.7% |
| Run 3 loss rate | 0% | 25.0% |
| Dominant failure | Missed blocks | Two-in-a-row threats not blocked |

**Core Finding:** Both approaches rapidly generate strategically correct statements ("block two-in-a-row threats"), yet prompt-only recursion struggles to convert those insights into reliable move-level execution. MAL bridges this "knowing-doing gap" by codifying stable patterns into deterministic procedures, achieving monotonic improvement with lower token costs once rules activate.

**Critical Distinction:** In reinforcement learning, knowledge is stored in model weights. In prompt-recursive learning, knowledge is stored in context. Neither phase modifies any weights.

## 1. Background and Related Work

This experiment sits at the intersection of external memory systems, agentic tool use, and inference-time scaffolding. Key research threads include:

### Memory Augmentation and Retrieval

- **Retrieval-Augmented Generation (RAG):** Combines parametric memory with retrieved documents. (Lewis et al. 2020)
- **kNN-LMs and RETRO:** Retrieval-augmented language modeling via nearest-neighbor datastores or large corpora. (Khandelwal et al. 2020; Borgeaud et al. 2022)
- **Agent memory hierarchies:** Systems managing memory tiers and context paging. (Packer et al. 2023 — MemGPT)

### Language Feedback Loops and Agentic Learning Without Weight Updates

- **Reflexion:** Improves agents using verbal reflection stored in episodic memory, not weight updates. (Shinn et al. 2023)
- **Generative Agents:** Structured memory plus reflection for long-horizon behavior in simulations. (Park et al. 2023)

### Tool Use, Reasoning-and-Acting, and Programmatic Scaffolds

- **ReAct:** Interleaves reasoning traces with actions querying external environments. (Yao et al. 2022)
- **Toolformer:** Trains language models to decide when to call external tools and incorporate results. (Schick et al. 2023)

### Long-Context Limitations and Recursive Language Models

- **Context degradation ("context rot"):** Performance degrades with increasing input length in non-uniform ways. (Chroma Research 2025)
- **RULER:** Benchmark extending needle-in-haystack tests into multi-hop and aggregation tasks, showing performance drops as context grows. (Hsieh et al. 2024)
- **Interactive reading:** Treating the model as an agent that decides how to read long contexts via iterative prompting (MemWalker). (Chen et al. 2023)
- **Recursive Language Models (RLMs):** An inference strategy treating long prompts as external environments, allowing programmatic decomposition and recursive sub-calls. (Zhang, Kraska, Khattab 2025)

### Where This Study Fits

MAL functions as a lightweight agent architecture: the LLM creates and updates natural-language memory, but deterministic code executes critical subroutines. Phase B is closer to "reflection-in-context" than to the full 2025 MIT CSAIL RLM framework.

## 2. Terminology Note

In recent literature, "Recursive Language Models (RLMs)" refers to an inference paradigm where an LLM treats the prompt as an external environment and can programmatically inspect, decompose, and recursively invoke sub-calls. This report's Phase B implements a simplified "prompt-recursive lesson injection" baseline, not the full MIT CSAIL RLM framework. See arXiv:2512.24601 for complete details.

## 3. Experimental Setup

### Task

Tic-Tac-Toe provides a controlled environment where optimal play is known and failure modes are classifiable. The learner always plays **O** and **moves first** -- a deliberate advantage. The opponent plays X (second) using **deterministic minimax** with a **10% random blunder rate**: on any move, there is a 1-in-10 chance the expert picks a random legal square instead of the optimal one.

**Research question:** Given first-move advantage and a near-perfect opponent with a known 10% weakness, can an LLM learn to reliably exploit that weakness through accumulated experience?

### Agent Testbed

A three-node KarnEvil9 swarm mesh runs repeated games and extracts lessons:

- **Learner agent (O):** Chooses moves; receives accumulated lesson text. Moves first.
- **Opponent agent (X):** Deterministic minimax policy with 10% random blunder.
- **Referee node:** Coordinates games, delegates moves, extracts lessons from transcripts.

### Phases

**Phase A: Memory-Assisted Learning (MAL)**

Uses an LLM for general reasoning and lesson generation. Introduces programmatic rules that bypass the model when conditions are met (immediate wins, mandatory blocks). Intent: convert reliable high-level insights into deterministic execution.

**Phase B: Prompt-Recursive Lessons (Simplified Baseline)**

Removes the programmatic rule layer. Every learner move is produced by the LLM, guided only by accumulated lesson text. The Phase B baseline deliberately used a stronger model (Claude Sonnet 4.5) to give the prompt-only approach the best possible chance.

### Controls

- Memory wiped between phases (Phase B starts with 0 lessons).
- Same opponent configuration (deterministic minimax + 10% random blunder).
- Same lesson extraction prompt.
- Learner always plays O and moves first.
- 12 games per run, lessons persist across runs within each phase.

### Evaluation Metrics

- Win/loss/draw counts per run.
- Loss rate (losses / 12 games) per run.
- Failure mode labeling for each loss (missed block, fork trap, etc.).

## 4. Results

### Phase A: Memory-Assisted Learning (MAL)

| Run | Lessons in | Wins | Losses | Draws | Loss rate |
|-----|-----------|------|--------|-------|-----------|
| MAL-1 (fresh) | 0 | 2 | 5 | 5 | 41.7% |
| MAL-2 | 12 | 3 | 1 | 8 | 8.3% |
| MAL-3 | 24 | 4 | 0 | 8 | 0% |

MAL reaches zero losses by run 3, coinciding with activation of deterministic block logic that prevents missed two-in-a-row defenses.

### Phase B: Prompt-Recursive Lessons Baseline

| Run | Lessons in | Wins | Losses | Draws | Loss rate |
|-----|-----------|------|--------|-------|-----------|
| B-1 (fresh) | 0 | 1 | 5 | 6 | 41.7% |
| B-2 | 12 | 2 | 6 | 4 | 50.0% |
| B-3 | 24 | 1 | 3 | 8 | 25.0% |

Note the non-monotonic trajectory: more lessons can help, but can also introduce noise or conflicting guidance.

### Head-to-Head Comparison

| Run | MAL loss rate | Prompt-recursive loss rate |
|-----|---------------|---------------------------|
| Run 1 (fresh) | 41.7% | 41.7% |
| Run 2 (12 lessons) | 8.3% | 50.0% |
| Run 3 (24 lessons) | 0% | 25.0% |

### Failure Mode Breakdown

| Failure Mode (losses only) | MAL occurrences | Prompt-recursive occurrences |
|---------------------------|-----------------|------------------------------|
| Failed to block a two-in-a-row threat | 4 (67%) | 9 (64%) |
| Fell into a fork trap | 1 (17%) | 3 (21%) |
| Poor positional play (edges over corners) | 1 (17%) | 2 (14%) |

## 5. Analysis and Interpretation

### The Knowing-Doing Gap

After only a few games, both systems produced lessons describing correct strategy: "block two-in-a-row threats immediately," "prioritize corners over edges," "avoid fork setups." The critical finding: the prompt-recursive baseline repeatedly failed at the simplest procedural task -- reliably detecting and blocking imminent threats.

A natural-language lesson like "always block when the opponent has two in a row" is strategically sound but algorithmically underspecified. It does not force an explicit scan over all eight winning lines or a deterministic board-state-to-block mapping. The lesson is strategic but not procedural. MAL reduces this gap by compiling strategy into deterministic control logic.

### Convergence Characteristics

MAL improves monotonically (5 -> 1 -> 0 losses), consistent with deterministic rules never regressing once enabled. The prompt-recursive baseline is non-monotonic (5 -> 6 -> 3 losses): adding more lesson text can introduce conflict, redundancy, or prompt drift -- consistent with broader long-context research showing performance degradation as context grows.

**Connection to Long-Context Research:** While this experiment is small-scale, the non-monotonic behavior aligns with findings from long-context evaluation work. See Chroma's Context Rot report and RULER benchmark.

### Purity vs Practicality

The prompt-recursive baseline is attractive for simplicity: no special tooling, no hand-written heuristics. However, in environments where correctness depends on exact symbolic checks (like threat detection), code-level execution provides a large reliability and compute win.

**Critical observation:** The prompt-recursive approach used Claude Sonnet 4.5 (significantly more capable than Haiku 4.5) yet underperformed MAL with Haiku. This suggests the bottleneck is not model capability -- it is the **representation gap** between natural-language lessons and procedural execution. A weaker model with code-level scaffolding outperforms a stronger model with only natural-language self-guidance.

## 6. Token Economics and Scaling Behavior

### Per-Move Token Growth

Both approaches inject lessons into the learner prompt, so input tokens grow approximately linearly with lesson count:

| Lessons in memory | Estimated input tokens per move | Where observed |
|------------------|--------------------------------|-----------------|
| 0 | ~300 | Run 1, game 1 |
| 12 | ~660 | Run 2, game 1 |
| 24 | ~1,020 | Run 3, game 1 |
| 36 | ~1,380 | Run 3, game 12 |

### Decomposing the Cost Advantage

Once deterministic rules activate, MAL reduces LLM calls per game by bypassing the model for forced tactical decisions:

| Metric | MAL (Run 3) | Prompt-recursive (Run 3) | Prompt-recursive (same model) |
|--------|-----------|--------------------------|------------------------------|
| LLM calls per game | ~3.5 | ~6 | ~6 |
| Total tokens per game | ~3,100 | ~5,800 | ~5,800 |
| Estimated per run (12 games) | ~37,000 | ~70,000 | ~70,000 |
| Model | Haiku 4.5 | Sonnet 4.5 | Haiku 4.5 |
| Estimated cost per run | ~$0.03 | ~$0.25 | ~$0.06 |

The headline comparison -- MAL at ~$0.03 vs prompt-recursive at ~$0.25 -- represents ~8x savings, but this conflates two independent variables:

1. **Architectural advantage (~2x):** MAL's programmatic rules bypass Claude for ~50% of moves. Code execution is always cheaper than LLM inference.
2. **Model pricing advantage (~4x):** MAL used Haiku ($0.80/M input) while Phase B used Sonnet ($3/M input). This was deliberate -- MAL's scaffolding compensates for a weaker model -- but represents a design choice.

The third column shows the apples-to-apples comparison: prompt-recursive on Haiku would cost ~$0.06 per run, making MAL only ~2x cheaper. The architectural token savings is real but modest. **The larger implication:** MAL enables cheaper models by offloading critical decisions to code, while prompt-recursive learning demands more capable (and expensive) models because every decision flows through the LLM.

### Scaling: Neither Has Infinite Context

A natural question: does prompt-recursive learning offer scaling advantages through "infinite context"? This is misconceived. **Both approaches are bounded by the context window**, and MAL actually scales better:

| Scaling property | MAL | Prompt-recursive |
|-----------------|-----|------------------|
| Knowledge in context | Only unpromoted lessons | All lessons (entire history) |
| Knowledge in code | Promoted rules (zero tokens) | None |
| Context pressure at scale | Low (lessons graduate to code) | High (all lessons stay in prompt) |
| Cost trajectory | Flattens as rules absorb patterns | Grows linearly with experience |

In MAL, high-confidence lessons are **promoted to deterministic code**, removing them from the context window entirely. MAL's context pressure _decreases_ as the system matures, while prompt-recursive context pressure only grows.

**Note:** Cost values are estimates depending on provider pricing, tokenization, and exact prompts. Use as order-of-magnitude guidance, not guaranteed billing.

## 7. Failure Modes

### Training Signals for Recursive Control

The experiment fixed recursion structure externally: 12 games per run, 3 runs per phase, one lesson per game. A full recursive system would need the model to control its own recursion -- deciding _when_ to recurse, _when_ to consolidate, and _when_ to stop. The data suggests three training signals:

**Lesson similarity as a halt signal:** By run 3, over 70% of lessons were restatements of "take opposite corners" and "block two-in-a-row." A system measuring semantic similarity could detect saturation. Signal: _if the last K lessons have cosine similarity > 0.85 with existing lessons, stop recursing and synthesize._

**Performance plateau as a depth signal:** Phase B run 2 performed worse than run 1 despite twice the lessons. A system monitoring its own win/loss rate could detect regression. Signal: _if loss rate hasn't decreased over N iterations, change the recursion strategy rather than continuing._

**Contradiction detection as a consolidation trigger:** Directly contradictory lessons accumulated: game 5 says "prioritize offense over defense," game 6 says "block before pursuing threats." Signal: _if lessons contain opposing directives for the same game state, recurse into consolidation before continuing play._

These signals point toward a **meta-recursive architecture**: the outer loop generates experience, an inner loop monitors quality, and a control layer decides whether to continue, consolidate, or terminate. The Phase B run 2 performance regression demonstrates what happens without these mechanisms.

### Looping Hallucinations

Self-referential feedback loops create a failure mode traditional RL systems avoid: **looping hallucinations**, where the model's incorrect outputs compound through iterations, creating degenerative spirals.

A mild form appeared in the data. The learner generated contradictory lessons across games 5-6: one advocating offense, the other defense. In subsequent games the model oscillated between them, unable to determine which applies because lessons lack conditional specificity.

**Degenerative loop:** Model produces incorrect analysis -> incorrect analysis becomes a lesson -> lesson biases future reasoning -> biased reasoning produces more incorrect analysis -> each iteration reinforces the error. Unlike RL, where reward signals allow correction, prompt-recursive learning has **no error-correction mechanism**. A bad lesson persists in context indefinitely.

In tic-tac-toe, this remained bounded. In open-ended domains (code generation, strategic planning, research synthesis), a looping hallucination could be catastrophic: the model confidently generates an incorrect pattern, encodes it as a lesson, and applies it to every subsequent task, each application generating new lessons that further reinforce the mistake.

**Mitigations:**

- **External validation:** Ground-truth checks on lesson accuracy before storage.
- **Lesson decay:** Reduce weight of older lessons, allowing the system to "forget" early mistakes.
- **Adversarial self-critique:** Ask the model to argue against each new lesson before acceptance.
- **Promotion to code (MAL's approach):** Code is verifiable in ways natural-language lessons are not.

## 8. Threats to Validity

- **Model mismatch across phases:** The prompt-recursive baseline used a stronger model (Sonnet) than MAL (Haiku). This makes the result conservative in MAL's favor on reliability but complicates attributing differences solely to architecture.
- **Small sample size:** 36 games per phase is enough to observe large effect sizes (like 0 vs non-zero losses), but not enough to precisely estimate variance or small differences.
- **Non-independence:** Lessons accumulate across runs within a phase, so runs are not independent trials.
- **Opponent stochasticity:** A 10% blunder rate adds variance; some wins/losses may reflect opponent noise rather than learner improvement.
- **Prior knowledge:** LLMs likely have pretraining exposure to Tic-Tac-Toe optimal strategy. Measured improvements concern _execution consistency_ given a prompt and scaffolding, not learning from scratch.

## 9. Lessons Learned

This toy domain makes failure modes legible. The experiments suggest practical insights about "learning without weight updates" in LLM agents:

- **Natural-language advice is not a procedure.** Agents often verbalize correct strategy while failing to execute corresponding scan-and-act routines. Reliability improves when strategy is compiled into explicit algorithms.
- **Memory helps until it becomes clutter.** Accumulating lessons can improve behavior but can introduce redundancy, conflict, and prompt drift. Without deduplication and consolidation, longer lesson buffers are non-monotonic in performance.
- **Deterministic promotion yields monotonicity.** Once a rule is expressed as code, it does not "forget" or regress, and it lowers variance from sampling.
- **Benchmarks must separate policy improvement from opponent randomness.** A blunder-rate opponent reasonably avoids draw-only play, but injects variance. Reporting random seeds, realized blunders, and confidence intervals matters even in small domains.
- **Terminology needs to match current literature.** "Recursive Language Models" has a specific meaning in recent long-context work. If the system is a lesson-injection loop rather than a full prompt-as-environment recursive decomposition framework, naming it explicitly prevents confusion.

**Design pattern:** Use the LLM to explore, hypothesize, and summarize. Use deterministic code to execute parts demanding perfect precision or exhaustive checks. Treat promotion-to-code as a reliability and cost optimization, not a retreat from learning.

## 10. What's Next

Current results are a useful mechanistic demonstration but not strong evidence about general learning dynamics. Valuable follow-ups fall into three buckets: tightening evaluation, strengthening the memory loop, and testing generality.

### Tighten the Evaluation

1. **Remove the model confound:** Run both conditions on the same base model (ideally on at least two models) to isolate architecture from capability.
2. **Add a true no-learning baseline:** Same prompts and model, but with lessons disabled and no deterministic rules.
3. **Increase sample size and control randomness:** Run more games per condition, fix opponent seeds, and report confidence intervals for loss rates and key failure modes.
4. **Measure tokens and latency:** Replace estimated token counts with measured input-output token logs per call, per game, and per run.

### Strengthen the Learning Loop

1. **Lesson consolidation:** Deduplicate and merge lessons into a small canonical set, with explicit conflict detection and versioning.
2. **Verification gates:** Before accepting a lesson, test it against a verifier or simulator. For Tic-Tac-Toe this can be exact: validate that a proposed rule never increases loss probability against minimax.
3. **Structured lessons:** Experiment with JSON schemas or checklists (for example, "scan all 8 lines, block if any has 2 opponent marks and 1 empty"). This tests whether failures stem from language ambiguity or attention/execution.
4. **Full RLM-style scaffolding:** Implement a prompt-as-environment REPL workflow and recursive sub-calls to align with modern RLM definitions in the literature.

### Test Generality Beyond Tic-Tac-Toe

1. **Harder games:** Small board games with larger state spaces and more tactical patterns (for example, Connect Four) stress both search and memory.
2. **Long-context tasks:** Apply the same MAL vs prompt-recursive baseline to long-document QA, codebase navigation, or deep-research benchmarks.
3. **Agentic workflows:** Evaluate whether promotion-to-code improves stability in multi-step tool-use tasks, where verbal plans often fail at execution details.

## 11. Reproducibility Checklist

**Code location:** [github.com/oldeucryptoboi/KarnEvil9](https://github.com/oldeucryptoboi/KarnEvil9)

**Experiment driver:** [scripts/tic-tac-toe-swarm.ts](https://github.com/oldeucryptoboi/KarnEvil9/blob/master/scripts/tic-tac-toe-swarm.ts)

- **Models:** Claude Haiku 4.5 (Phase A), Claude Sonnet 4.5 (Phase B).
- **Opponent policy:** Deterministic minimax + 10% random blunder rate.
- **Runs:** 3 runs per phase, 12 games per run.
- **Lesson persistence:** Within-phase accumulation (0 -> 12 -> 24 lessons).
- **Key prompts:** System prompt for the learner, and the post-game lesson extraction prompt.
- **Random seeds:** Record and fix seeds for opponent blunders and any sampling.
- **Logging:** Store full game transcripts and lessons per game.

When publishing results, also include: exact model versions, temperature/top-p settings, max tokens, and a reproducible environment lockfile.

## 12. References

1. Zhang, A. L., Kraska, T., Khattab, O. (2025). _Recursive Language Models_. arXiv:2512.24601.
2. Lewis, P. et al. (2020). _Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks_. arXiv:2005.11401.
3. Khandelwal, U. et al. (2020). _Generalization through Memorization: Nearest Neighbor Language Models_. ICLR 2020. arXiv:1911.00172.
4. Borgeaud, S. et al. (2022). _Improving Language Models by Retrieving from Trillions of Tokens_. ICML 2022. arXiv:2112.04426.
5. Packer, C. et al. (2023). _MemGPT: Towards LLMs as Operating Systems_. arXiv:2310.08560.
6. Shinn, N. et al. (2023). _Reflexion: Language Agents with Verbal Reinforcement Learning_. arXiv:2303.11366.
7. Park, J. S. et al. (2023). _Generative Agents: Interactive Simulacra of Human Behavior_. arXiv:2304.03442.
8. Yao, S. et al. (2022). _ReAct: Synergizing Reasoning and Acting in Language Models_. arXiv:2210.03629.
9. Schick, T. et al. (2023). _Toolformer: Language Models Can Teach Themselves to Use Tools_. arXiv:2302.04761.
10. Chroma Research (2025). _Context Rot: How Increasing Input Tokens Impacts LLM Performance_.
11. Hsieh, C.-P. et al. (2024). _RULER: What's the Real Context Size of Your Long-Context Language Models?_ arXiv:2404.06654.
12. Chen, H. et al. (2023). _Walking Down the Memory Maze: Beyond Context Limit through Interactive Reading_. arXiv:2310.05029.
