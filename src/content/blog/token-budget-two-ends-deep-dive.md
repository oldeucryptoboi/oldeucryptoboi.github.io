---
title: "Two Ends of the Token Budget: Caveman and Tool Search"
description: "Caveman compresses what the model says. Tool search defers what the API sends. Two opposite strategies for one budget: the context window."
pubDate: 2026-04-11
tags: ["ai", "claude", "agents", "tokens"]
---

Every Claude Code session has a single budget: the context window. Two hundred thousand tokens, give or take, that have to hold the system prompt, the tool definitions, the conversation history, the user's input, the model's output, and (if extended thinking is on) the chain of thought. There is exactly one pile, and everything gets withdrawn from it.

The pile has two openings. Tokens flow in from the system side: tool schemas, system prompt, prior turns, files the model read. And tokens flow out from the model side: explanations, code, commit messages, plans. Both sides count against the same total. Both sides eat budget.

Two projects look at this single budget from opposite ends.

The first is **Caveman**, a Claude Code plugin that makes the model talk like a caveman. "Why use many token when few do trick." The mechanism is a prompt that tells the model to drop articles, filler, hedging, and pleasantries while keeping technical substance intact. The README claims ~75% output token savings, the benchmark table averages 65% across ten real tasks, and a bonus tool called `caveman-compress` rewrites your `CLAUDE.md` so the model reads less every session start. ([github.com/JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman))

The second is **tool search**, a system inside Claude Code that defers MCP tool definitions until they're needed. When a session connects three MCP servers with 50 tools each, that is 60,000 tokens of schema overhead before the conversation starts. Tool search hides the schemas behind a discovery tool, lets the model search for what it needs, and loads only the matching definitions. Same context space, fewer tokens spent on tools the model never calls. (Already documented in [tool-search-deep-dive.md](./tool-search-deep-dive.md).)

Both projects target the same number — total tokens consumed per session. They reach it from opposite ends. Caveman compresses what the model says. Tool search defers what the API sends. One is lossy and lives at the prompt layer. The other is lossless and lives at the API layer. One is a single skill file plus two hooks. The other is a multi-stage pipeline with snapshot survival across compaction.

This article walks both systems in enough detail to reconstruct them, then compares the trade-offs. Where the savings come from. What gets sacrificed. Which side of the budget you should attack first. And whether you can run them at the same time. The point is not to crown a winner — they don't compete, they compose. The point is to understand the budget well enough to spend it on purpose.

---

## Where the tokens actually go

Look at a typical Claude Code session and label every token by source. A rough breakdown for an active coding session with a couple of MCP servers connected:

```
SYSTEM PROMPT                ~3,000 tokens   (1.5%)
TOOL DEFINITIONS             ~25,000 tokens  (12.5%)   <- built-ins + MCP
PROJECT MEMORY (CLAUDE.md)   ~2,000 tokens   (1%)
CONVERSATION HISTORY         ~80,000 tokens  (40%)     <- grows over time
TOOL OUTPUTS (file reads)    ~50,000 tokens  (25%)
MODEL OUTPUT (this turn)     ~5,000 tokens   (2.5%)
HEADROOM                     ~35,000 tokens  (17.5%)
-----------------------------------------------
TOTAL                        200,000 tokens
```

Numbers vary by session, but the shape is consistent. Three categories dominate: tool definitions, conversation history, and tool outputs. Model output is small per turn but large per session, and it is the only category that grows even when the model is doing nothing useful — every "Sure, I'd be happy to help with that" is paid for.

Now color the categories by who controls them:

- **System controls**: system prompt, tool definitions, project memory loaded at start.
- **User controls**: the prompts they type, the files they ask Claude to read.
- **Model controls**: its own output.
- **Conversation history**: a slow-burning mix of all three, accumulating over turns.

Caveman attacks one cell of this grid: model output. It can also attack project memory via `caveman-compress`. Tool search attacks another cell: tool definitions. Neither touches the conversation history directly — that is compaction's job, and it is a different article.

The interesting observation is that the two projects aim at the smallest dominant category each. Tool definitions are ~12% of the budget. Per-turn model output is ~2.5%. Why bother?

Because of the per-turn cost. Tool definitions are sent on **every** API call. A single 60,000-token tool block, multiplied by 50 API calls in a session, is 3 million input tokens — and input tokens, while cheaper than output, are not free. Model output, similarly, is sent every turn and accumulates into the conversation history, where it costs input tokens forever after. A 1,000-token explanation early in a session pays its full price once on output, then keeps re-paying as input on every subsequent turn.

The right way to think about both savings is per-turn, amortized:

```
caveman_savings_per_session  ~ avg_response_tokens * turns * compression_ratio
tool_search_savings_per_turn ~ deferred_tool_tokens * turns_until_discovered
```

Caveman's savings scale with conversation length. Tool search's savings scale with the number of unused tools. A session with 50 turns and a chatty model wins big on caveman. A session with 200 MCP tools and a 5-tool workflow wins big on tool search. A session with both wins on both.

The categories don't fight for the same byte of budget. They fight for the same total.

---

## Caveman: compress what you say

Caveman is a Claude Code plugin. It ships as a marketplace package you install with one command:

```bash
claude plugin marketplace add JuliusBrussee/caveman
claude plugin install caveman@caveman
```

The installer puts three things in your environment: a SKILL file, two hooks, and several sub-skills (`caveman-commit`, `caveman-review`, `caveman-compress`). The mechanism is, at its core, a prompt. Not a parser, not a token filter, not a fine-tuned model. A prompt.

### The skill file

The main skill file opens with frontmatter declaring trigger phrases ("caveman mode", "talk like caveman", "less tokens", "be brief") and then lays out the rules in a few hundred tokens. The rules are blunt:

```
Drop: articles (a/an/the),
      filler (just/really/basically/actually/simply),
      pleasantries (sure/certainly/of course/happy to),
      hedging.

Fragments OK.
Short synonyms (big not extensive,
                fix not "implement a solution for").
Technical terms exact.
Code blocks unchanged.
Errors quoted exact.

Pattern: [thing] [action] [reason]. [next step].
```

Then a before/after pair so the model has a concrete example to imitate:

```
NOT: "Sure! I'd be happy to help you with that.
      The issue you're experiencing is likely caused by..."
YES: "Bug in auth middleware.
      Token expiry check use `<` not `<=`. Fix:"
```

That is the entire compression engine. The model reads the rules, the pattern, and the example, then applies them to its own output. There is no postprocessor. There is no validator. The model is doing the work.

### Intensity levels

The skill defines six levels along a single axis: how much grammar to keep.

| Level | Effect |
|-------|--------|
| `lite` | Drop filler and hedging. Keep articles and full sentences. Professional but tight. |
| `full` | Drop articles, fragments OK, short synonyms. The default. |
| `ultra` | Abbreviate (DB, auth, cfg, req, res, fn). Strip conjunctions. Use arrows for causality. One word when one word suffices. |
| `wenyan-lite` | Semi-classical Chinese. Drop filler. |
| `wenyan-full` | Full classical Chinese. Subjects often omitted. Classical particles. |
| `wenyan-ultra` | Maximum classical compression. |

The wenyan modes are not a joke. Classical Chinese is one of the most token-efficient written languages ever invented; most tokenizers handle CJK characters as one to two tokens each, and a wenyan sentence often packs the meaning of an English paragraph. The README's example for "Why does the React component re-render?" goes from 41 English tokens (lite) down to about 9 wenyan-ultra tokens. Same answer.

### The hooks

Two small Node scripts wire the skill into Claude Code's hook system.

`caveman-activate.js` runs on `SessionStart`. It writes a flag file at `~/.claude/.caveman-active` containing the current mode (`full` by default), and prints a short ruleset reminder to stdout. Stdout from a `SessionStart` hook becomes part of the session's context, so the model sees the rules even before it reads the user's first prompt.

```
on session_start:
    mkdir ~/.claude
    write ~/.claude/.caveman-active = "full"
    print to stdout:
        "CAVEMAN MODE ACTIVE.
         Drop articles/filler/pleasantries/hedging.
         Fragments OK. Pattern: [thing] [action] [reason].
         Code/commits/security: write normal."
```

`caveman-mode-tracker.js` runs on `UserPromptSubmit`. It reads the user's input from stdin, looks for `/caveman` slash commands, parses the level argument, and rewrites the flag file. It also recognizes "stop caveman" and "normal mode" as deactivation phrases:

```
on user_prompt_submit:
    prompt = read_stdin().lower().trim()

    if prompt starts with "/caveman":
        cmd, arg = split first two words
        case cmd:
            "/caveman-commit"   -> mode = "commit"
            "/caveman-review"   -> mode = "review"
            "/caveman-compress" -> mode = "compress"
            "/caveman":
                case arg:
                    "lite"         -> mode = "lite"
                    "ultra"        -> mode = "ultra"
                    "wenyan-lite"  -> mode = "wenyan-lite"
                    "wenyan"       -> mode = "wenyan"
                    "wenyan-ultra" -> mode = "wenyan-ultra"
                    default        -> mode = "full"
        if mode set:
            write ~/.claude/.caveman-active = mode

    if prompt matches "stop caveman" or "normal mode":
        delete ~/.claude/.caveman-active
```

The flag file is mostly cosmetic: a separate statusline script reads it to display a `[CAVEMAN:ULTRA]` badge in the UI. The skill itself is what tells the model how to talk.

### Auto-clarity

The skill carves out scenarios where compression hurts more than it helps:

- Security warnings (the user must see the threat).
- Irreversible action confirmations (the user must understand what they're approving).
- Multi-step sequences where reading order matters.
- The user is confused.

In these cases the model is told to drop caveman, write normally, then resume. The example in the skill:

```
> Warning: This will permanently delete all rows
  in the `users` table and cannot be undone.
> ```sql
> DROP TABLE users;
> ```
> Caveman resume. Verify backup exist first.
```

This is a soft guardrail — the model's judgement decides when "irreversible" or "confused" applies. The skill provides the rule; the model interprets it.

### caveman-compress

The bonus sub-skill turns the compression on a different file: your `CLAUDE.md`. Project memory loads on every session start, so its size is paid every time you launch Claude. `caveman-compress` rewrites your memory file in caveman style and keeps the human-readable version as a `.original.md` backup:

```bash
/caveman:compress CLAUDE.md
```

```
CLAUDE.md           # compressed (Claude reads this every session)
CLAUDE.original.md  # human-readable backup (you read and edit this)
```

The README's table reports 35–60% compression on real memory files, average 45%. The trick is the same: drop prose, keep code blocks, URLs, file paths, commands, and version numbers verbatim. The compressed memory file is still valid Markdown; the model parses it the same way. The human just has to translate when they want to update it (which is what the original backup is for).

### The benchmark

Caveman's headline number is "~75% output token savings." The benchmark table in the repo measures real Claude API token counts across ten tasks and reports an average of 65%, with a range from 22% (a refactor task that is already terse) to 87% (a verbose explanation task). The repo also cites a March 2026 paper that found brevity constraints can *improve* accuracy on certain benchmarks ([arxiv.org/abs/2604.00025](https://arxiv.org/abs/2604.00025)) — the relevant claim is that asking large models to be brief doesn't necessarily make them dumber and sometimes makes them sharper.

The README is also honest about the limit: caveman only affects output tokens. Thinking/reasoning tokens are untouched. A model with extended thinking enabled still pays the same internal monologue cost. Caveman makes the *mouth* smaller, not the brain.

The whole system is roughly a hundred lines of JavaScript and sixty lines of skill prompt. It works because the model is the engine.

---

## Tool search: defer what you receive

Tool search is the opposite shape: a multi-stage pipeline inside Claude Code that keeps tool definitions out of the API request until the model proves it needs them. No prompt to the model that says "use fewer tools." No instruction at all. The model gets a smaller tool list, full stop, and a way to ask for more.

### The deferral decision

Tools are classified as deferrable or always-on. The classifier is a priority checklist, walked top to bottom on every tool every request:

```
function is_deferred_tool(tool):
    # Explicit opt-out from the tool author
    if tool.always_load:
        return false

    # MCP tools are deferred by default
    if tool.is_mcp:
        return true

    # ToolSearch itself is the bootstrap, never deferred
    if tool.name == "ToolSearch":
        return false

    # FORK_SUBAGENT carve-out: when the fork-subagent variant
    # of Agent is enabled, Agent stays loaded so the model can
    # spawn subagents without a discovery hop
    if feature("FORK_SUBAGENT") and tool.name == "Agent":
        if fork_subagent_enabled():
            return false

    # KAIROS carve-out: the Brief tool is always loaded under
    # KAIROS because it is the primary user-facing channel
    if (feature("KAIROS") or feature("KAIROS_BRIEF"))
            and tool.name == BRIEF_TOOL_NAME:
        return false

    # KAIROS + REPL carve-out: SendUserFile stays loaded when
    # the REPL bridge is active, because the model needs to
    # push files synchronously without a search round-trip
    if feature("KAIROS") and tool.name == SEND_USER_FILE_TOOL_NAME
            and is_repl_bridge_active():
        return false

    # Built-ins opt in by setting the should_defer flag
    return tool.should_defer
```

The order matters. `always_load` is checked first so a tool author can declare something too important to defer. MCP comes next because most MCP tools are not used per-session. ToolSearch is exempt because the model needs at least one tool to discover the others. Then three feature-flagged carve-outs handle special cases where a discovery hop would break a primary workflow: spawning subagents, the user-facing Brief channel, and synchronous file sends through the REPL bridge. Built-in tools the model uses every turn (file read, bash, edit) fall through to the final `should_defer` check, which they leave false.

### The threshold check

There are three modes, resolved from the `ENABLE_TOOL_SEARCH` environment variable:

- `tst` — always defer the deferrable tools. The default.
- `tst-auto` — defer only when the deferred tool tokens exceed a threshold. The threshold is set as `tst-auto:NN` where `NN` is the percentage.
- `standard` — never defer.

There is also a kill switch one level up: if `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` is set, the mode resolver returns `standard` immediately and a separate field-stripping pass on the API request removes `defer_loading: true` from any schema that still carries it. This is the escape hatch for users on enterprise contracts that pin against beta features.

The auto threshold defaults to 10% of the context window. For a 200K-token model, the cutoff is 20,000 tokens. If the deferred tools would have cost less than 20K, deferral is disabled and everything loads — no point in paying the discovery latency for a small saving.

The token count itself comes from the API's count-tokens endpoint when available, falling back to a character-per-token heuristic (about 2.5 chars per token) when the endpoint is unreachable. There is also a per-tool overhead constant (around 500 tokens) that gets subtracted before comparing the per-tool cost against the threshold, because the count-tokens endpoint reports each tool's full request envelope. The heuristic is intentionally conservative — it slightly overestimates, biasing toward enabling deferral, because the cost of over-deferring (one extra search turn) is much smaller than the cost of under-deferring (60K tokens of unused schema per request).

```
threshold = floor(context_window * 10 / 100)
deferred_token_count = count_tokens_or_heuristic(deferred_tool_schemas)
enabled = (deferred_token_count >= threshold)
```

There is one more gate, an optimistic disable that fires before any of the above. If the user has not explicitly set `ENABLE_TOOL_SEARCH` and the API base URL points at a non-Anthropic endpoint (a proxy or gateway), tool search returns `false` from its optimistic check and the ToolSearch tool is not even registered. The reasoning is that proxies often mediate beta headers in unpredictable ways, and silently sending `defer_loading` to a gateway that strips it would mean the model gets the bare-name list with no way to discover tools. Better to disable cleanly than fail mysteriously.

The mode also affects model selection. A model name allowlist (defaulting to a hardcoded list with `haiku` as the only entry, but live-overridable through a remote config flag named `tengu_tool_search_unsupported_models`) marks specific models as not yet tool-search-capable. When the active model matches a pattern on that list, tool search returns `standard` regardless of the env var. The remote-config indirection exists so that newly released models can be flipped on or off without a Claude Code release.

### The search tool

When deferral is on, the model sees a `ToolSearch` tool in its tool list. The deferred tools are listed by name in the system prompt with a one-liner each (an A/B test on richer search hints in the listing was retired in early 2026; the current build sends just the names), but their full schemas — where the bulk of the tokens lives — are absent.

The model searches in three forms, plus a couple of operators:

```
ToolSearch({ query: "github create issue" })             // keyword search
ToolSearch({ query: "select:mcp__github__create_issue" }) // direct selection
ToolSearch({ query: "select:read_file,write_file,bash" }) // multi-select
```

The first is a keyword search across tool names and descriptions, scored against an internal hint field and returning the top-N matches (default 5, settable via `max_results`). The second is a direct selection by exact name, used when the model already knows what it wants — there is also a fast path that handles a bare tool name as an implicit select. The third is a comma-separated multi-select that loads several tools in a single turn, which the model uses when it has decided up front that a workflow needs three or four tools together.

The keyword form supports two operators. A `+` prefix on a term marks it as required (`+github +issue create` will not match a tool that lacks "github" or "issue" in its searchable text). A `mcp__server__` prefix on a query is recognized as a server-scoped search and only ranks tools from that MCP server. Everything else is a regular optional term that contributes to the score but does not gate the match.

All three forms return `tool_reference` content blocks — opaque pointers that the API expands into full tool definitions on the next request:

```
{
  "type": "tool_reference",
  "tool_name": "mcp__github__create_issue"
}
```

That is a few dozen tokens to mark a tool as discovered. On the next turn, the API sees the reference, looks up the full schema (the request itself still flags the tool with `defer_loading: true`, but discovery overrides deferral on the API side), and includes the schema in the tool list sent to the model. The model now has the schema and can call the tool normally.

The beta header that opts an API request into all of this differs by provider. On the first-party Anthropic API the header is `advanced-tool-use-2025-11-20` and goes in the `betas` field. On Bedrock and Vertex it is `tool-search-tool-2025-10-19` and on Bedrock specifically it goes in `extraBodyParams` instead of `betas`, because Bedrock's request envelope handles betas differently. The provider check happens in the request builder, after deferral is decided but before the request is signed.

### The discovery loop

Across turns, the system maintains a set of "discovered" tools by scanning the conversation history for `tool_reference` blocks. The tool list sent to the API on each turn is the union of:

```
sent_tools = always_on_tools
           + ToolSearch
           + (deferred_tools intersected_with discovered_in_history)
```

A tool that was discovered on turn 5 stays in the tool list for turns 6 onward, because its `tool_reference` is still in the message history. The model doesn't need to re-discover it. The system reads the history every turn and rebuilds the discovered set.

### Surviving compaction

The tricky case is context compaction. When the conversation gets too long, Claude Code summarizes earlier turns into a compressed history. The summary doesn't preserve raw `tool_reference` blocks — they are metadata, not text.

Tool search handles this with a snapshot. Before compaction runs, the system writes the current discovered tool set into a boundary marker that survives the summary. After compaction, the discovery loop reads the boundary marker first, then continues scanning the post-compaction history. Tools discovered before the compaction boundary stay discovered.

```
on compaction:
    snapshot = current discovered tool set
    write snapshot to compaction boundary marker

on discovery loop:
    discovered = snapshot from boundary marker (if present)
              + tool_references in post-boundary history
```

Without the snapshot, every compaction would force the model to re-discover its workflow. The user would notice as a sudden surge of `ToolSearch` calls right after compaction.

### The fail-closed hint

One last detail. The discovery loop is best-effort — there are scenarios where the model tries to call a tool whose schema is not in the current request. It might remember the tool from a long-ago turn whose `tool_reference` got summarized away. It might hallucinate a tool name. It might fire a deferred tool right after a snapshot loss. In every case, the failure happens before the API call: Claude Code validates the model's tool input against a Zod schema on the client, and the schema for a deferred-but-undiscovered tool was never sent to the API in the first place, so the model is emitting parameters blind. Untyped parameters from a model that hasn't seen the schema almost always fail Zod's parse — strings where numbers were expected, missing required fields, wrong array shapes.

Claude Code catches the Zod error, formats it into a tool-result block, and then asks one extra question: was this an undiscovered deferred tool? The check has four parts:

```
1. Is tool search optimistically enabled at all?
2. Is the ToolSearch tool actually in the current tool list?
3. Is this tool a deferred tool?
4. Is this tool's name absent from the discovered set?
```

If all four are true, the formatted error gets a hint appended to it before being returned to the model:

```
"This tool's schema was not sent to the API —
 it was not in the discovered-tool set derived
 from message history. Without the schema in your
 prompt, typed parameters (arrays, numbers, booleans)
 get emitted as strings and the client-side parser
 rejects them. Load the tool first: call ToolSearch
 with query 'select:<tool_name>', then retry this call."
```

The hint is not an API error interception. It is an augmentation of a *client-side* validation failure, layered on top of the Zod report so the model sees both the parser's complaint and the meta-explanation for why the parser is unhappy. The model reads the combined message, calls ToolSearch with a direct selection, gets the schema, and retries on the next turn. One extra turn instead of a conversation-ending failure, and zero risk of leaking anything to the API — the failed call never went out.

### What it costs

The savings: a session with 200 MCP tools and a 5-tool workflow drops from ~90,000 input tokens of tool definitions per turn to ~15,000 (the always-on tools plus ToolSearch plus the 5 discovered). Across 20 turns, that is 1.5 million input tokens saved.

The cost: one extra API turn per discovery (call ToolSearch, get the reference, then call the actual tool on the next turn). For a workflow that calls 5 distinct tool groups, that is 5 extra turns over a 20-turn session — 25% more API calls, but each call is dramatically cheaper. The math works out heavily in favor of deferral.

The risk: the model can't find a tool it needs because the search didn't surface it. The keyword search and the fail-closed hint both exist to mitigate this. In practice the failure mode is "model takes one extra turn to search differently," not "model gives up."

The whole system is significantly more code than caveman. It is a parser for the deferral mode environment variable, a model-name allowlist with remote-config override, a proxy gateway optimistic disable, a token counter with caching and a heuristic fallback, a content-block emitter, a discovery loop scanning history, a snapshot mechanism for compaction survival, a Zod error augmenter for the fail-closed case, and (in the fullscreen UI environment, gated behind an `is_fullscreen_env_enabled` check) a collapse rule that absorbs ToolSearch calls silently into the surrounding tool group so the user never sees the discovery hop. It is lossless, by which I mean the model gets exactly the same schema it would have gotten without deferral — just delivered later.

---

## Lossy versus lossless

Here is the cleanest way to see the difference: caveman is lossy, tool search is lossless.

Caveman makes the model write less. The tokens that disappear are real characters of real meaning — articles, hedges, transitional phrases, polite framing. A model running caveman cannot say "Sure, I'd be happy to help with that" because the rules forbid it. The savings come from content the model would otherwise produce. The savings are *content reduction*.

Tool search makes the API send fewer tool definitions. The tool definitions that disappear from a given API call are not lost forever — they are reachable via discovery. A model running tool search and a model running standard mode receive the *same* schema for any tool they actually call. The only difference is when the schema arrives. The savings come from definitions the model never asked about. The savings are *delivery deferral*.

The implication is different failure modes.

Caveman fails by *misjudging compression*. The skill says "drop articles, except when the user is confused." But who decides when the user is confused? The model. And the model has to decide on every response. The auto-clarity carve-out exists because compression can mask important nuance. A security warning written in caveman might miss the severity. A multi-step procedure written in fragments might be misread out of order. The skill puts the rule in front of the model and trusts the model's judgement to apply it. When the judgement is right, the user reads a tighter, clearer answer. When it is wrong, the user reads a fragment that omits a precondition and they have to follow up. The wrong call is a content quality issue, not a system failure — there is no exception thrown, no error logged, just an answer that was too compressed.

Tool search fails by *missing a search hit*. The model needs `mcp__github__create_issue` and searches for "github issue create." If the search ranking is good, the right tool is in the top 5 results. If not, the model tries another query, or fails to find the tool and the user has to disambiguate. The fail-closed hint catches the worst case — calling a not-yet-loaded tool — and converts it to a one-turn detour. The wrong call is a *latency* issue, not a correctness issue. The tool the model eventually loads is the same tool it would have gotten without deferral.

This is the asymmetry that matters: **caveman trades correctness margin for tokens; tool search trades latency for tokens.**

If you can afford to lose a little correctness margin in exchange for big output savings, caveman pays. If you can afford to wait one extra API round-trip in exchange for big input savings, tool search pays. The two things you can lose are different, so the projects don't compete — they complement each other.

There is a second asymmetry worth naming. Caveman's output reduction is *sticky*: every compressed response stays in the conversation history forever, so the savings compound. A 1,000-token explanation reduced to 250 tokens saves 750 tokens once on output and another 750 tokens of input on every future turn that includes it. Tool search's input reduction is *per-turn*: a deferred tool that costs 500 tokens saves 500 tokens on every API call where it is not discovered. Both compound in their own way, but caveman's compounding is one-shot-then-permanent while tool search's compounding is ongoing-while-relevant.

Caveman's failure case shows up immediately (the user sees a confusing fragment). Tool search's failure case shows up immediately (the model takes an extra turn). Both projects fail loudly, which is the right kind of failure — silent wrong answers are the dangerous ones.

A useful mental model: caveman is a lossy codec, tool search is a lazy loader. Lossy codecs trade fidelity for size. Lazy loaders trade latency for size. They are both compression; they are compressing different things, and they are paying with different currencies.

---

## When each pays off

Both projects have a sweet spot. Knowing which side of the budget your session leans on is the first question. The answer depends on the workload.

### Caveman wins when

- **Output is a meaningful share of the token bill.** Long explanations, design discussions, debugging walkthroughs, architectural Q&A. Anywhere the model produces paragraphs.
- **A human reads the output.** Caveman's compression is optimized for human readers — fragments, abbreviations, arrow notation. Tools that parse model output (linters, JSON consumers, automation hooks) might choke on caveman style. The skill exempts code blocks, commits, and PR titles for exactly this reason.
- **The conversation is long.** Caveman's savings compound through history. A 50-turn session with 65% output compression doesn't just save 65% on each response; it saves 65% on the input cost of every subsequent turn that includes those responses.
- **You are paying per output token and want the bill smaller.** Output tokens are typically the most expensive line on the invoice. Cutting them in half halves the most expensive line.

Caveman loses when the model is mostly producing code or structured output, because those are exempt. A session that is 90% file edits and 10% explanations wins very little from caveman.

### Tool search wins when

- **You have a lot of MCP tools.** Three servers with 50 tools each. A custom server with 200 endpoints. Anything where the schema cost is measured in tens of thousands of tokens.
- **You only use a small fraction of them per session.** A workflow that touches 5 tools out of 200 is the ideal case. A workflow that touches 150 of 200 wastes the discovery overhead.
- **Sessions are long.** Discovered tools stay discovered for the whole session (and across compactions, via the snapshot). The discovery cost is paid once.
- **You are paying per input token and tool definitions are a meaningful share of input.** Per-turn API cost has tool definitions as a big cell; deferring them shrinks every turn.

Tool search loses when the tool surface is small or the workflow uses most tools. A session with one MCP server and a 10-tool workflow that touches all 10 has nothing to gain — the deferred tools would all be discovered immediately.

### When to use both

Most non-trivial Claude Code sessions will benefit from at least one of them and some will benefit from both. The decision is empirical. Run a session with measurement on (the API returns token counts in usage) and look at the breakdown:

```
input_tokens
  |- system + tool defs    <- target with tool search
  |- memory (CLAUDE.md)    <- target with caveman-compress
  |- conversation history  <- compounded by caveman
  +- tool outputs          <- target with read planning
output_tokens
  +- model responses       <- target with caveman
```

If the system + tool defs cell is the biggest, install tool search (it is already on by default in modern Claude Code; just check it is not disabled). If model responses are the biggest, install caveman. If both are big, install both. If neither is big, you don't have a problem.

The wrong move is to install compression aggressively without knowing where the bleed is. Compression has costs (correctness margin, latency, complexity). Pay them where they earn back.

---

## Stacking them

The two projects compose because they live at different layers and target different parts of the budget.

```
            +--------------------------+
USER  --->  | /caveman:compress        |   compresses CLAUDE.md
            |  CLAUDE.md               |   (input, system layer)
            +----------+---------------+
                       |
                       v
            +--------------------------+
            | Claude Code session      |
            |                          |
SYSTEM ---> |  tool search             |   defers tool schemas
            |  (deferral pipeline)     |   (input, API layer)
            |                          |
MODEL  ---> |  caveman skill           |   compresses responses
            |  (prompt + hooks)        |   (output, prompt layer)
            +----------+---------------+
                       |
                       v
                  API request
```

Three different compression points in the same pipeline:

1. **`caveman-compress` rewrites CLAUDE.md.** This is a one-time, user-triggered batch operation. It runs before Claude Code starts and shrinks the project memory file the agent will load on every session. The savings are paid once and collected on every future startup. Layer: filesystem. Currency: prose tokens dropped permanently.

2. **Tool search defers MCP schemas.** This runs inside Claude Code on every API request. It decides which tool definitions to send and which to mark as deferred. Layer: API request builder. Currency: schema tokens delayed (sent later, when the model calls a discovered tool, or never if the model never asks).

3. **The caveman skill compresses model responses.** This is a prompt the model reads at session start and obeys on every turn. Layer: model output. Currency: response tokens dropped permanently.

None of the three steps interfere with each other. The compressed `CLAUDE.md` is still valid Markdown — Claude reads it the same way it reads any memory file. Tool search operates on the API request after the system prompt and memory are assembled, so a compressed memory file just means fewer tokens to ship alongside fewer tool definitions. The caveman skill operates on the model's outgoing tokens, which are downstream of everything the API sent in. The three layers stack cleanly.

A session with all three running might look like:

```
Without compression:    200K tokens used over 30 turns
With caveman-compress:  198K tokens used (memory shrunk)
   + tool search:       170K tokens used (tool defs deferred)
   + caveman skill:     130K tokens used (output halved, history compounds)
```

The numbers depend wildly on the workload, but the structure is real: the three savings accumulate because they target three non-overlapping cells of the budget.

This is the design payoff. The token budget is one number, but it has internal structure. Different compression strategies attack different cells. A project that aims at the right cell can win an order of magnitude more than a project that aims at a cell already being squeezed by something else. The two ends of the pipe — input and output — are not competing for the same byte. They are collaborating on the same budget.

---

## Closing

Claude Code, like every LLM agent, runs against a context window. The window is finite. Every category that shares it — tool schemas, memory, conversation history, model output — pays from the same pool. This sounds like a single-knob optimization problem until you look at where the tokens actually go, and then it becomes a multi-cell budget where each cell has its own dynamics, its own controllers, and its own compression strategy.

Caveman attacks one cell from one direction: compress the model's outgoing tokens by giving the model a stricter style guide. The mechanism is a prompt. The cost is correctness margin at the edges, mitigated by an auto-clarity carve-out. The savings compound through conversation history. The implementation is roughly a hundred lines of JavaScript and sixty lines of skill prompt — you could read the whole thing in ten minutes.

Tool search attacks a different cell from a different direction: defer MCP tool schemas until they are searched and discovered. The mechanism is an API content block (`tool_reference`) plus a discovery loop that scans history. The cost is one extra API turn per discovered tool group, mitigated by a fail-closed hint that catches the worst case. The savings are per-turn and amortize over long sessions. The implementation is significantly more code, with snapshot survival, threshold logic, mode flags, and UI hiding.

The two projects are not competing for the same byte. Caveman compresses output. Tool search defers input. They live at different layers — one is a prompt the model reads, the other is a request builder the model never sees. They can run at the same time and the savings combine.

The shared lesson is the one that is easy to miss: before you compress anything, look at the budget. The right compression strategy depends on which cell is actually leaking tokens. Measure first. Compress second. Caveman would say: budget broken? look. fix biggest leak. then next.

---

**Sources**

- Caveman: [github.com/JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman)
- "Brevity Constraints Reverse Performance Hierarchies in Language Models" (March 2026): [arxiv.org/abs/2604.00025](https://arxiv.org/abs/2604.00025)
- Tool search deep dive: [tool-search-deep-dive.md](./tool-search-deep-dive.md)
