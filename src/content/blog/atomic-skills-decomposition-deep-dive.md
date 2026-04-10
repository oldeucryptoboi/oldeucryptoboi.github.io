---
title: "Five Atomic Skills, Two Approaches: Claude Code and a Paper"
description: "An arXiv paper trains coding agents on five atomic skills. Claude Code achieves the same with runtime sub-agents. Two of the five are missing entirely."
pubDate: 2026-04-10
tags: ["Claude Code", "Anthropic", "AI Agents", "Coding Agents", "AI Engineering"]
---

# Five Atomic Skills, Two Approaches: Claude Code and a Paper

## The Paper's Claim

In late 2025, a paper appeared on arXiv arguing that the way the field trains coding agents is broken. The standard recipe — fine-tune a base model on SWE-bench-style end-to-end repair traces — produces models that look strong on the benchmark and fall apart everywhere else. The paper is *Atomic Skills Decomposition for Coding Agents* (Ma et al., [arXiv:2604.05013](https://arxiv.org/abs/2604.05013)). Its central proposal is to stop training on composite tasks entirely. Instead, decompose what a coding agent actually does into five irreducible skills, generate training data for each skill in isolation, and train them jointly with reinforcement learning so the model learns each skill against a clean, narrow reward signal.

The five skills the paper picks are:

1. **Code Localization** — given a bug report, find the file and function that need to change.
2. **Code Editing** — given a target location and a description, produce the patch.
3. **Unit-Test Generation** — given code, produce tests that exercise it correctly and reject mutations.
4. **Issue Reproduction** — given a bug report, write a script that fails before the patch and passes after.
5. **Code Review** — given a diff, produce a binary judgment that matches a held-out human label.

The paper's training rig is austere. It gives the model two tools: `bash` and `str_replace`. That's it. No grep tool, no glob tool, no file-read tool, no agent-spawning tool, no MCP, no skills. Everything the model wants — search, navigation, file inspection, test runs — has to go through bash. The reward functions are equally austere: exact-match for localization (+1 if the predicted file/function set matches ground truth, –1 otherwise), all-tests-pass for editing, mutation-survival for test-gen, failure-flip for reproduction, label-agreement for review. The infrastructure is K8s with 25,000+ Docker images and 10,000+ concurrent sandboxes. The base model is GLM-4.5-Air-Base (106B total, 12B active). The reported gain is **+18.7% average** over the composite-trained baseline across held-out benchmarks.

If you read the paper and then use Claude Code for an afternoon, the contrast is jarring. Claude Code is the *opposite* design. It exposes dozens of tools instead of two. It ships several built-in sub-agents instead of a single inference loop. It has three different code-review slash commands, each with a multi-step orchestration plan, false-positive filtering, parallel sub-agents, and remote-execution fleets. And yet — and this is the interesting part — when you go looking for the paper's *other* four skills, two of them are missing entirely. There is no unit-test-generation agent. There is no issue-reproduction agent. The asymmetry is sharp enough to tell you something about which problems are bottlenecked at inference time and which are bottlenecked elsewhere.

This article walks the comparison layer by layer. First the tool surface — why Claude Code went the opposite direction from `bash + str_replace`. Then the sub-agent architecture — how Claude Code does at *inference* time what the paper does at *training* time. Then the five skills, mapped one by one against Claude Code's actual surface. Then the gaps, which turn out to be the most interesting part. Then the over-developed review pipeline, which has more machinery than the other four skills combined. Finally, the reward-hacking parallels — both systems fail-closed, but against opposite threat models.

The thesis: **the paper decomposes at training time so the model learns clean primitives. Claude Code decomposes at inference time so the user can compose primitives. Both are valid. They produce wildly different system architectures.**

---

## Layer 1: The Tool Surface

The paper gives the model two tools and lets it discover everything else through bash:

```
# Paper's tool surface, in full:
bash(command: string) -> { stdout, stderr, exit_code }
str_replace(path: string, old: string, new: string) -> ok | error
```

That's the entire interface. If the model wants to find a function definition, it runs `grep -rn "def foo" .`. If it wants to read a file, it runs `cat path/to/file`. If it wants to find files matching a pattern, it runs `find . -name "*.py"`. If it wants to run tests, it runs `pytest -xvs path/to/test`. There is no `read_file` tool, no `glob` tool, no `grep` tool. The reasoning is explicit in the paper: a narrow tool surface forces the model to learn general bash skill, which transfers across environments. A model that knows how to use grep against an unfamiliar codebase is more useful than a model that knows how to call a custom `search_code` API.

Now look at Claude Code. The visible tool surface (before MCP, before skills) is wide: there's an Agent tool for dispatching sub-agents, a Bash tool, dedicated Glob and Grep tools, a FileRead, a FileEdit, a FileWrite, a NotebookEdit, a WebFetch, a WebSearch, a TodoWrite, an AskUserQuestion, a Skill tool, plan-mode tools, MCP-resource tools, and more. The shipped surface is on the order of dozens of tools, not two.

And the model is actively *steered away from bash* for things bash could trivially do. Watch a Claude Code session and you'll notice the pattern: when the model wants to read a file, it calls the dedicated read tool instead of `cat`. When it wants to find files, it calls the dedicated glob tool instead of `find`. When it wants to search content, the dedicated grep tool instead of raw `grep`. When it wants to edit, the dedicated edit tool instead of `sed`. The shell route exists, but it's the fallback, not the default.

This is the opposite of the paper's design philosophy. The paper says: *force the model to use bash so it learns bash.* Claude Code says: *steer the model away from bash so the user can review what the model did.* The reasons converge on something like UX. When the model writes `sed -i 's/foo/bar/g' main.py`, the user sees an opaque shell command. When it writes `Edit({ file: "main.py", old: "foo", new: "bar" })`, the user sees a structured diff in the terminal. The dedicated tool isn't faster or smarter than `sed` — it's *legible*. A user reviewing tool calls in a terminal scrollback wants every operation framed and named, not piped through a shell.

The trade-off is real. The paper trains a model that gets *better* at bash. Claude Code trains a model (well, prompts a model) that gets *better at picking the right specialized tool*. The Claude Code approach assumes the model is already strong enough at bash that you can pull it off the bash path without losing capability — and that you'd rather have legibility. The paper assumes you're starting with a weaker base model and training matters.

There's a second axis. The paper's narrow tool surface is also a precondition for its training procedure to converge: rewards can be local to the final answer, not to which tool the model picked at each step. Claude Code isn't training on its own traces — it uses a frozen base model and shapes behavior with the prompt — so it can afford a wide surface. Two systems, two consistent positions. Notice what each one is optimizing for.

---

## Layer 2: Sub-Agents as Atomic Skills

The paper trains the model on each atomic skill in isolation. At inference time, the trained model can perform any of the five skills, switching between them within a single conversation. There is no "localization mode" the model enters and leaves — the skill boundaries exist only during training.

Claude Code does the inverse. It exposes sub-agent boundaries at *inference time*. When the main model wants to perform a focused task, it calls the Agent tool with a `subagent_type` argument and that spawns a child conversation with a different system prompt, a different tool subset, possibly a different model, and an isolated transcript. The child runs to completion and returns a single message back to the parent. The parent never sees the child's intermediate turns.

Here's the round-trip in pseudocode:

```
# Parent model emits a tool call:
tool_call = Agent(
    subagent_type = "Explore",
    description   = "find auth middleware",
    prompt        = "Search for express middleware that validates JWTs..."
)

# Conceptually, the dispatcher does this:
def call_agent_tool(args, parent_context):
    spec  = look_up_agent(args.subagent_type)        # e.g. the Explore profile
    if not allowed_by_permissions(spec, parent_context):
        return error("agent not allowed")

    # Build a child context with a narrowed surface.
    child = fork_context(parent_context,
        system_prompt    = spec.system_prompt,
        tools            = restrict_tools(parent_context.tools, spec),
        model            = pick_model(spec, parent_context),
        drop_project_md  = spec.is_read_only,        # CLAUDE.md not needed
        drop_git_status  = spec.is_read_only,
        isolated_log     = True,                     # separate transcript
    )

    # Run the child to completion in its own loop.
    final_message = ""
    for turn in run(child):
        # Intermediate turns go to the isolated transcript, NOT the parent.
        if turn.is_final:
            final_message = turn.text

    return tool_result(final_message)

# The parent only ever sees `final_message`. The dozens of grep/read
# turns the child took to find the answer never enter the parent's context.
```

The contrast is precise. The paper compresses skills into one model that can switch between them; Claude Code compresses each skill's *intermediate work* by sandboxing it in a child context whose only output is a summary message. The paper compresses by training a smaller behavioral surface. Claude Code compresses by running the wide surface inside a quarantine.

Several sub-agents are available out of the box. There's an **Explore** agent — read-only, fast, optimized for searching and reading code. There's a **Plan** agent — read-only, designed to produce structured implementation plans. There's a **Verification** agent — explicitly adversarial, told to try to break the implementation it was handed. There's a **general-purpose** agent — the catch-all when the parent wants a sub-conversation but doesn't fit the other shapes. And there are a couple of narrow helpers (a docs-lookup agent that knows where to find Claude Code's own documentation, a tiny one for editing the user's statusline config) that have nothing to do with the paper's five skills — they're domain-specific affordances for working *with* Claude Code itself.

Notice the shape. Three of the agents (Explore, Plan, Verification) are bound directly to phases of a software-engineering workflow: *find the code*, *plan the change*, *check the change broke nothing*. One is the catch-all. The rest are domain-specific helpers.

The Explore agent, in particular, looks like the paper's localization skill rendered as a runtime construct. Its instructions cast it as a file-search specialist in strict read-only mode: it can glob, grep, and read, but it cannot create, modify, delete, move, or even use shell redirects to write a file. The restriction isn't enforced by polite request — the file-mutation tools are literally absent from its tool list. If the model inside the child tries to call one, the dispatch fails before any API request is made. This is the same trick the paper plays with reward shaping — give the skill a narrow surface so its only path to success is doing the thing it was named after — except the enforcement happens at tool dispatch time instead of at gradient-update time.

Two more details matter. The fast read-only agents drop project-level instructions (CLAUDE.md) from their child context entirely — a search agent hunting for a function signature doesn't need the project's "use bun, not npm" rule, and at the scale these agents are spawned, dropping a 5–15KB instruction blob from every spawn adds up. They also strip the parent's git-status preamble, which can be tens of kilobytes of stale diff data.

The pattern: a built-in sub-agent is a *narrowed inference context* with a focused prompt, a restricted tool list, a possibly-different model, and aggressive context omission. This is what the paper calls "atomic skill" — but constructed at inference time and dispatched into from a parent that decides when each skill is needed.

---

## Layer 3: Mapping the Five Skills

Now the comparison can be precise. For each of the paper's five atomic skills, what does Claude Code have?

### Skill 1: Code Localization → the Explore agent

The paper's localization task: given a natural-language bug description, produce a set of `(file, function)` tuples that need editing. The reward is exact-match against ground truth.

Claude Code's analog is the Explore agent. The match is strong. Explore is read-only, optimized for speed (it runs on a fast/cheap model rather than the parent's main model), focused entirely on search and navigation, and returns a final message that the parent uses to decide where to edit. The parent's natural call pattern is:

```
# Parent's reasoning (semantically, in the model's head):
"User reported the login button doesn't work. I need to find the login
button handler before I can fix it."

tool_call: Agent({
  subagent_type: "Explore",
  description: "find login button handler",
  prompt: "Search the codebase for the login button handler. Look for
           'login' in component files, identify which component renders
           the button, and trace the click handler to its implementation.
           Return the file path and function name."
})

# Explore runs a dozen Glob/Grep/Read calls internally.
# Returns: "The login button is rendered in the LoginForm component
#          inside the auth components directory. Its click handler is
#          handleSubmit, which calls authClient.signIn from the auth
#          service module."

# Parent now has the location. Proceeds to editing.
```

The match isn't perfect. The paper's exact-match reward forces the model to be precise rather than enumerate. Claude Code's Explore can return ten files when one would do, with no penalty — it's actively nudged toward thoroughness rather than terseness. The training-time reward forces concision; the runtime prompt forces breadth. Two design philosophies for the same skill, derived from how they get measured.

### Skill 2: Code Editing → the Edit tool, not an agent

The paper's editing task: given a target location and a description, produce a patch and have the test suite pass. The reward is binary pass/fail.

Claude Code's analog is *not* an agent. It's the Edit tool itself:

```
# Claude Code's editing surface, semantically:
Edit({
  file_path: "auth/login.py",
  old_string: "if len(password) < 8:",
  new_string: "if len(password) < 12:",
  replace_all: false
})
# -> validates that old_string occurs exactly once
# -> applies the substitution
# -> returns the updated file region
```

There is no "editing agent." The edit happens directly in the parent context. This is significant because it shows how Claude Code treats the editing skill: editing doesn't get a focused sub-context. The parent already knows what to edit (it just got the location from Explore), and the edit should be visible in the parent's transcript so the user can see and review every change.

The closest thing to "editing-as-a-skill" in Claude Code is the Plan agent, which produces a structured implementation plan ending with an enumeration of the files the parent should change. Plan isn't editing — it's *prescription* for editing. The actual edit is deferred to the parent.

Why the asymmetry with Explore? Because edits change the world. A search agent that does its own grep deep inside a sub-context produces a string the parent can choose to act on. An editing agent that does its own writes inside a sub-context produces *changed files* the parent has to discover by re-reading, and the user can't see what changed without going hunting for it. Editing stays in the parent because *side effects are global*. Localization can be quarantined because *its only output is text*.

### Skill 3: Unit-Test Generation → nothing

The paper's test-gen task: given an existing function, produce unit tests that pass on the original implementation and fail on mutated versions of it. The reward is the rate at which the tests catch a generated mutation suite.

Claude Code's analog: there is none.

There's no "test-gen" sub-agent. There's no test-gen slash command. The bundled skills cover things like verifying, debugging, simplifying, getting unstuck, looping, and remembering — but no test generator. The closest thing is the Verification agent's general instruction to "run the project's test suite" — which is *running existing tests*, not generating new ones.

Test generation is structurally hard for an inference-time agent because the reward signal is a *future* property: tests are good if they catch future mutations or regressions, neither of which exist when the test is being written. The paper can use mutation testing as a reward because mutation suites can be generated mechanically at training time. At runtime, there is no mutation suite — just a function the user wants tests for, and a vague hope the generated tests are useful. Claude Code punts: the model writes tests inline with Edit/Write, no specialized prompting, no evaluation. The implicit assumption is that if you want good tests, you'll review them yourself.

### Skill 4: Issue Reproduction → also nothing

The paper's reproduction task: given a bug report, write a script that fails before the patch and passes after. The reward is `failure(pre) ∧ ¬failure(post)`.

Claude Code's analog: also none, but with a twist.

There's no reproduction agent. There's no `/reproduce` slash command. But there *is* a piece of the Verification agent's playbook that does part of the work: when the change being verified is a bug fix, the Verification agent's strategy says, in effect, "reproduce the original bug, verify the fix, run regression tests, check related functionality for side effects." Reproduction is folded into verification.

That folding has consequences. Verification runs *after* a fix has been applied, only for bug-fix tasks, and is optimized for *checking the fix worked* — not for *demonstrating the bug exists* before there's a fix. The paper's reproduction skill is forward-looking (write a repro to anchor a future fix). Claude Code's is backward-looking (write a repro to prove the fix landed). The forward-looking version doesn't exist as a sub-agent — if a user asks Claude Code to "first reproduce this," the parent handles it ad hoc with the same general-purpose tools it uses for everything else, with no specialized prompt.

### Skill 5: Code Review → over-developed (see Layer 5)

Code review is the one skill where Claude Code has *more* infrastructure than the paper. So much more that it gets its own section. Briefly: there are at least three review surfaces (`/review`, `/ultrareview`, `/security-review`), each with its own orchestration plan, sub-agent fan-out, false-positive filtering, and remote-execution architecture. Layer 5 walks through them.

### The shape of the mapping

Tally it up:

```
Paper's skill        | Claude Code's analog              | Strength
---------------------|-----------------------------------|----------
Code Localization    | Explore agent                     | strong
Code Editing         | Edit tool (no agent)              | tool only
Unit-Test Generation | (none)                            | absent
Issue Reproduction   | (folded into Verification agent)  | partial
Code Review          | /review, /ultrareview, /security  | over-built
```

The pattern is striking. Two skills are missing as runtime constructs. Two are present but in shapes that don't map cleanly to the paper. One is wildly over-developed. If you drew a Pareto frontier of "runtime infrastructure invested per skill," it would not look like the paper's evenly-trained five-way decomposition. It would look like a long tail.

---

## Layer 4: Two Gaps

The two gaps — test generation and issue reproduction — are the most informative part of this comparison, because they show where Claude Code went out of its way *not* to build a sub-agent. The absences are not oversights.

### Why no test-gen agent

Three reasons. First, **the reward signal is delayed**: a test is good if it catches future mutations or regressions, and neither exists at runtime. The agent can write tests that pass against the current implementation, but "passes" is trivial to satisfy (`assert True` passes). The hard part is "would catch a real bug," and there's nothing in the runtime context to grade against.

Second, **good tests are project-specific**. They use the project's framework, fixtures, mocks, and naming conventions. A test-gen sub-agent would need to load all of that — which is the opposite of what sub-agents are for. They *strip* context to stay focused. A test-gen agent that drops CLAUDE.md and project conventions would produce tests that look right and fail to integrate.

Third, **the user is the wrong audience**. When the paper trains a test-gen skill, the consumer of the tests is the model itself, in a self-improvement loop. When Claude Code generates tests, the consumer is a human developer who has to read every test and decide whether to commit it. An autonomous test-generator that produces 30 tests in a sub-context and returns a summary ("generated tests for the auth module") is *worse* than the parent producing two well-named tests inline that the user can see.

So Claude Code lets the parent handle test writing the same way it handles any other writing task: with Edit/Write, in full view of the user. The agent boundary would hurt more than help.

### Why no issue-reproduction agent

Reproduction has a different problem: **the reproduction is the bug report**. When a user comes to Claude Code with a bug, they usually already have the repro — it's in the message they typed. "I click the login button and nothing happens." "When I run `npm test`, it fails with TypeError." The repro is the input, not the output.

The paper's repro task assumes the input is a bug report from a tracker that may or may not contain a runnable repro. The model has to construct one. That's meaningful in a *batch* setting where the model is grading itself against a corpus of issues. It's much less meaningful in an *interactive* setting where the user is at the terminal and can be asked clarifying questions. Claude Code's parent handles repro by reading the description, asking follow-ups if needed, running the failing command in Bash, and observing — no sub-agent because no need for context isolation.

### What this asymmetry tells us

The two gaps line up around a single principle: **a sub-agent makes sense when the work is search-shaped or check-shaped, not when it's create-shaped**. Search (Explore, Plan) explores a large space and returns a small answer. Check (Verification) probes a target and returns a verdict. Both benefit from quarantine — they generate intermediate noise the parent doesn't need.

Create — writing code, writing tests, writing repros — does the opposite. It produces output the parent and the user want to see in full. Quarantining it inside a sub-context hides the very thing the user came for. The paper doesn't have to make this distinction because it isn't optimizing for legibility — it's optimizing for a frozen reward function during training. Once the model is trained, there's no parent and no quarantine. Claude Code, with a frozen base model and a runtime architecture, has to decide which work belongs in which scope, and the decision falls cleanly along search-vs-create lines.

---

## Layer 5: The Over-Developed Review

The fifth skill, code review, is where Claude Code has *more* infrastructure than the paper. Three different review surfaces ship out of the box, each with its own design.

### `/review` — the simple local path

The simplest entry point is `/review`. It's a slash command that produces a prompt for the parent model to execute directly:

```
# /review's prompt, semantically:
You are an expert code reviewer. Follow these steps:

1. If no PR number is provided, run `gh pr list` to show open PRs
2. If a PR number is provided, run `gh pr view <number>` to get details
3. Run `gh pr diff <number>` to get the diff
4. Analyze the changes and provide a thorough code review including:
   - Overview of what the PR does
   - Code quality and style
   - Specific suggestions
   - Potential issues or risks

Focus on: correctness, project conventions, performance, test coverage,
security considerations.
```

This is a prompt-only command. No sub-agent, no fan-out, no special tools — the parent uses Bash + Read to run the gh commands and produce the review. It's the bash-and-str_replace philosophy of the paper applied to one slash command. The hard part — the review judgment — is pushed entirely to the model's prior.

### `/security-review` — the three-step orchestration

`/security-review` is more ambitious. Its prompt is a multi-page document with hard exclusion rules, precedents, severity guidelines, confidence scoring, and explicit orchestration:

```
# /security-review, semantically (the orchestration block):
Begin your analysis now. Do this in 3 steps:

1. Use a sub-task to identify vulnerabilities. Use repository exploration
   tools to understand context, then analyze the PR for security
   implications. Include all of the categories, exclusions, and precedents
   in the sub-task prompt.

2. Then for each vulnerability identified by step 1, create a new
   sub-task to filter false positives. Launch these as PARALLEL sub-tasks.
   Include the FALSE POSITIVE FILTERING instructions in each.

3. Filter out any vulnerabilities where the sub-task reported confidence < 8.

Your final reply must contain the markdown report and nothing else.
```

This is fan-out-fan-in. The parent dispatches one sub-task to find candidate vulnerabilities. For each candidate, it dispatches another sub-task in parallel, asking it to grade confidence on a 1–10 scale. Then it filters by threshold. The orchestration is *in the prompt*, not in code — the parent is told the algorithm and trusted to follow it.

The hard exclusions are the interesting part. The prompt enumerates 18 specific things that are *not* vulnerabilities (DOS, log spoofing, regex injection, race conditions without concrete impact, dependency outdatedness, memory safety issues in Rust, unit-test files, SSRF that only controls the path, etc.) plus 12 precedents. These look like the paper's reward shaping but applied via prompt: the model is told what *not* to flag, because the cost of false positives is high. There's no learned reward function here — just a list hand-written by humans who triaged real security review reports and noticed patterns of overcalls. This is what reward shaping looks like when you don't get to train.

### `/ultrareview` — the remote fleet

`/ultrareview` is the heaviest. It doesn't run review in the user's local Claude Code session at all. It teleports the work to a remote container — Claude Code on the web — and runs a *fleet of agents* in parallel against the same diff. The published behavior tells you the shape: it takes roughly 10–20 minutes, runs in the cloud, costs against a quota with overage billing, and notifies the local session when findings are ready. Inside that envelope, multiple agents run in parallel against the same diff for around twenty-odd minutes. The orchestrator collects findings, dedupes them, and pushes the result back. There's a precondition check before launch: if the diff against the merge-base is empty, it bails before spinning up the container. There's a quota gate that decides whether the run is free, billed as overage, or refused.

Compare this to test-gen and reproduction, which have *zero dedicated infrastructure*. A fleet of agents reviewing a diff for twenty minutes is the high end of the long tail. The asymmetry is intentional: **review is the place where extra inference compute pays off**, because:

- The user has limited time to manually review code, so spending machine compute is a clear win.
- False positives are actionable (the user dismisses them) without breaking anything.
- True positives are high-value (they catch bugs before merge).
- Review is *parallelizable* — multiple agents on the same diff produce more findings, not multiple copies of the same finding (with dedup).

Test generation and reproduction don't have these properties. Generating five sets of tests in parallel doesn't give you better tests; it gives you five sets of tests you have to manually pick from. Reproducing a bug five times doesn't give you a better repro. So compute-heavy review pays off and compute-heavy test-gen doesn't.

This is a real lesson about which skills benefit from sub-agent decomposition. **Decomposition pays when the skill is parallelizable, the false-positive cost is low, and the user lacks time to do it manually.** Some of the paper's skills satisfy this; others don't; and Claude Code's investment lines up almost exactly with which skills do.

---

## Layer 6: Reward Hacking, in Reverse

Both systems worry about the model exploiting its environment to fake success. They worry about it differently.

The paper's reward-hacking concern is about training. The model is being optimized against a reward signal, and any path to a higher reward that doesn't involve actually solving the task is a path the optimizer will find. So the paper's training infrastructure does things like:

```
# Paper's reward-hacking countermeasures (from the paper):
- Strip the .git directory from every sandbox before training
  (otherwise the model learns to read git log for the answer)
- No network access in the sandbox
  (otherwise the model learns to fetch the patch from GitHub)
- Mutation generation is deterministic and held out
  (otherwise the test-gen reward leaks the mutation set)
- Code review labels are LLM-judged and the judge is held out
  (otherwise the review reward becomes "match a known judge")
```

The threat model: a *training* loop running thousands of times where the model gets to keep gradient updates from any successful trajectory. Every information leak in the sandbox becomes a shortcut.

Claude Code has the opposite threat model. It's running a single user's session, in their terminal, on their machine, with their files and their credentials. The model isn't being trained on the trajectory — it's executing a user request. The risk isn't the model reward-hacking *its own* training. The risk is the model *taking actions the user didn't authorize*, possibly because the user's input was crafted by an attacker (a malicious file the model read, a poisoned web page it fetched, a shell snippet it was asked to evaluate). The countermeasures live at *inference* time, in the tool layer. The visible behavior:

- **The bash analyzer asks before running anything ambiguous.** Run a bash command Claude Code doesn't fully recognize and you'll get a permission prompt rather than an automatic approval. The default is "I don't understand this command, can I run it?" not "looks fine to me."
- **Permission rules can allow, deny, or ask.** Tools and command patterns can be scoped per project. Deny rules always fire and cannot be overridden by the model's confidence.
- **The model is steered away from raw shell into named, framed tools** for read/edit/glob/grep, so every operation appears in the transcript with a clear name and inputs.
- **Read-only sub-agents simply can't call edit tools.** When the user spawns a search-shaped sub-agent, edit tools aren't merely discouraged in the prompt — they're absent from the child's tool list. There's no bypass through clever prompting.
- **Sub-agent intermediate work stays in an isolated transcript.** A misbehaving sub-agent can't poison the parent's reasoning by running away in its own context, because the parent only sees the final message it returns.

Both systems are fail-closed. Both have the principle that an unfamiliar construct should be asked-about rather than approved. But the *direction* of the failure mode is opposite:

- The paper fails closed against the model's optimizer finding shortcuts in training data.
- Claude Code fails closed against the model running attacker-influenced commands in production.

One is "the model is the attacker, the reward function is the victim." The other is "the user is the victim, the model is a vector." Same shape, opposite directions.

There's a third symmetry. Both systems carefully control **what the model knows about its evaluator**. The paper hides the mutation suite and the judge LLM from the model so it can't game them. Claude Code's `/security-review` hides the expected findings and instead hands the model 18 hard-exclusion rules and 12 precedents — negative space that defines the evaluator without revealing the answer key. Both systems have figured out that telling the model "these are the criteria you'll be judged on" produces a model that satisfies the criteria literally and misses the spirit.

---

## Closing

Two systems, five skills, opposite design philosophies. The paper decomposes at training time and produces a single trained model with five clean primitives. Claude Code decomposes at inference time and produces a runtime architecture where some primitives become sub-agents (Explore, Verification), some stay in the parent (Edit), some get folded into other skills (Reproduction inside Verification), and some don't exist (Test-Gen).

The interesting thing is that the absences are not bugs. They're consistent with a single principle: **a sub-agent is the right shape when the work is search-or-check and the output is a small judgment, and the wrong shape when the work is creation and the output is something the user wants to see in full.** Localization is search → sub-agent. Editing is creation → tool. Verification is check → sub-agent. Test-gen is creation → no sub-agent. Reproduction (forward-looking) is creation → no sub-agent. Review is parallelizable check → multi-agent fleet. The pattern holds.

The paper's contribution, viewed from the Claude Code side, is the demonstration that *training* can decompose a coding agent into clean primitives if you can construct the right reward functions. Claude Code's contribution, viewed from the paper's side, is the demonstration that *runtime* can decompose a coding agent into clean primitives if you accept that some skills don't decompose well at runtime and shouldn't be forced.

Neither approach is universally right. They're complements. A model trained the paper's way and deployed in Claude Code's runtime would, plausibly, be stronger than either alone — the trained skills would give the runtime sub-agents better priors, and the runtime decomposition would let the user see and steer creation work that training-time decomposition can't expose.

If you're building a coding agent, the lesson is to **decide which skills you're going to decompose and where you're going to put the seam**. Training-time decomposition needs cheap clean reward signals and tolerates an opaque inference loop. Runtime decomposition needs cheap clean *context boundaries* and tolerates a model that's already strong. Pick the one whose constraints match the system you can actually build. Or, like the paper plus Claude Code, do both — but at different layers.

Sources:

- *Atomic Skills Decomposition for Coding Agents*, Ma et al., [arXiv:2604.05013](https://arxiv.org/abs/2604.05013)
- Claude Code observable behavior: Explore, Plan, Verification, and general-purpose sub-agents; the `/review`, `/ultrareview`, and `/security-review` slash commands; the tool surface visible to the model in a normal session.
