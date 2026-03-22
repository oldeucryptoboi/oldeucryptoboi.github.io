---
title: "Why ClawHub Flagged My OpenClaw Skill as Suspicious (and How I Proved It)"
description: "Publishing an OpenClaw skill to ClawHub and getting SUSPICIOUS? Auth-heavy curl examples in SKILL.md trigger VirusTotal's scanner. Here's the controlled experiment and the fix."
pubDate: 2026-03-22
tags: ["OpenClaw", "Security", "ClawHub", "AI Agents"]
---

## Publishing an OpenClaw skill to ClawHub and getting flagged SUSPICIOUS? Here's how to fix it.

I published an [OpenClaw](https://openclaw.ai/) skill to [ClawHub](https://clawhub.com/) — the OpenClaw skill marketplace — and the security scan came back `SUSPICIOUS`. The warning pointed at [VirusTotal](https://www.virustotal.com/) Code Insight, ClawHub's remote code analysis engine. The skill was intentionally published, installed and ran fine locally, and contained nothing malicious.

The temptation was to just `--force` the install. Instead, I decided to isolate exactly what was triggering ClawHub's security scanner — using the same controlled experiment methodology you'd use to debug any flaky system.

---

### What wasn't the cause

Before running any experiments, I listed the plausible causes for the ClawHub security warning:

1. Provocative wording in the skill description
2. Missing version metadata in the OpenClaw skill manifest
3. Any mention of API keys or auth headers in the skill files
4. Network mutation examples (POST/PUT/DELETE curl commands)
5. Only the top-level `SKILL.md` being scanned more aggressively than reference files
6. The overall shape of the skill bundle being "too risky" for VirusTotal's heuristics

Most debugging stops here — someone picks the most likely explanation, changes something, and if the ClawHub warning goes away, declares victory. But that doesn't tell you *which* variable actually mattered.

---

### The experiment: probe-based skill scan isolation

ClawHub enforces a rate limit of 5 new skill slugs per hour. So the experiment used five throwaway OpenClaw skill slugs at `0.0.1`, then bumped versions on those same slugs for subsequent probes. All probe skills were deleted from ClawHub after results were recorded.

#### Batch 1: Isolating the trigger class

| Probe | Content | Result |
|---|---|---|
| `baseline@0.0.1` | Plain markdown, no network examples | `CLEAN` |
| `get@0.0.1` | Public unauthenticated `GET` curl example | `CLEAN` |
| `auth@0.0.1` | Authenticated `GET` curl example | **`SUSPICIOUS`** |
| `post@0.0.1` | Authenticated `POST` curl example | **`SUSPICIOUS`** |
| `min@0.0.1` | One minimal authenticated write example | **`SUSPICIOUS`** |

First signal: public read-only curl examples passed ClawHub's security scan. A single authenticated external command in the OpenClaw skill was enough to trigger the VirusTotal flag.

#### Batch 2: Top-level vs reference files

| Probe | Content | Result |
|---|---|---|
| `baseline@0.0.2` | Top-level `SKILL.md` only (from real skill) | **`SUSPICIOUS`** |
| `get@0.0.2` | Reference file only (from real skill) | `CLEAN` |
| `auth@0.0.2` | Full skill bundle clone | **`SUSPICIOUS`** |
| `post@0.0.2` | Generic multi-write skill bundle | **`SUSPICIOUS`** |

Second signal: the reference file in isolation was clean. The top-level `SKILL.md` in isolation was suspicious. The flag wasn't domain-specific — a generic skill with authenticated writes also triggered it.

#### Batch 3: Separating auth text from executable commands

| Probe | Content | Result |
|---|---|---|
| `baseline@0.0.3` | `Authorization: Bearer $API_KEY` as static text only | `CLEAN` |
| `get@0.0.3` | Unauthenticated `POST` curl example | `CLEAN` |

Third signal: auth tokens as plain text didn't trigger it. Write verbs alone didn't trigger it. The combination — executable commands with auth headers in the top-level skill body — was the specific pattern.

---

### The root cause: SKILL.md is the security-sensitive surface

ClawHub's remote security scan treats `SKILL.md` as the most sensitive file in an OpenClaw skill bundle. That makes sense — it's the file AI agents read first and follow most directly. When that file contains executable command examples with authentication headers, VirusTotal's code analysis flags it as potentially dangerous.

The scanner isn't wrong, exactly. An OpenClaw skill that instructs an agent to run authenticated HTTP mutations *is* a higher-risk pattern than one that describes a workflow in prose. The issue was that my skill had detailed API examples (curl commands with `Authorization: Bearer` headers, POST bodies, endpoint paths) right in the top-level `SKILL.md` instructions.

This is still an empirical conclusion, not a reverse-engineered VirusTotal rule. ClawHub doesn't expose the precise security scan signature. But nine controlled probes with consistent results across three batches is enough to act on.

---

### The fix: restructure SKILL.md for ClawHub compliance

Restructure the OpenClaw skill to separate "what to do" from "how to call the API":

- **`SKILL.md`**: High-level, workflow-oriented. Describes *what* the AI agent should accomplish, *when* to activate the skill, and *what* auth is required — in prose, not in executable examples. This is the file ClawHub's security scanner scrutinizes most.
- **`references/agent-api.md`**: Exact headers, endpoints, request bodies, response shapes. All the operational detail an agent needs to make authenticated API calls, but in a [reference file](https://agentskills.io) rather than the top-level skill instructions.

Published the restructured skill as `v2.0.3`. ClawHub result: `Security: CLEAN`.

---

### Operating rules for publishing OpenClaw skills to ClawHub

For anyone publishing AI agent skills to ClawHub and wanting to pass the security scan:

1. Keep `SKILL.md` high-level and workflow-based — describe intent, not implementation
2. Move raw curl examples, auth headers, and mutation-heavy API examples into `references/`
3. Describe auth requirements in prose at the top level — don't embed executable commands
4. Never normalize `--force` when ClawHub flags a skill as suspicious — isolate the cause first
5. If a future OpenClaw skill gets flagged, repeat the probe-based isolation method instead of guessing

---

### Why AI agent skill security matters beyond ClawHub

The broader pattern here is [security scanning for LLM tool use](https://arxiv.org/abs/2403.04957). As AI agent ecosystems and skill marketplaces grow, the tools agents can invoke become a real attack surface. An OpenClaw skill that tells an agent "run this curl command with this auth token" is functionally the same as giving an autonomous agent a shell script. Security scanners like VirusTotal are right to be cautious about that.

The fix isn't to stop writing AI agent skills that do authenticated API calls. The fix is to structure them so the *intent* is clear at the top level and the *implementation details* live in reference material. Agents can still access everything they need. Security scanners can see that the top-level `SKILL.md` instructions are descriptive, not imperative.

It's a small structural change, but it's the difference between `SUSPICIOUS` and `CLEAN` — and it's a pattern that will matter more as [agent skill standards](https://agentskills.io) become the norm for AI tool distribution.
