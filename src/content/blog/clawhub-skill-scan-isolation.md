---
title: "Why ClawHub Flagged My OpenClaw Skill as Suspicious (and How I Proved It)"
description: "Auth-heavy curl examples in SKILL.md trigger VirusTotal's code scanner. Here's the controlled experiment that proved it."
pubDate: 2026-03-22
tags: ["OpenClaw", "Security", "ClawHub", "AI Agents"]
---

<!-- seo_title: Why ClawHub Flagged My OpenClaw Skill as Suspicious -->
<!-- seo_description: Auth-heavy curl examples in SKILL.md trigger VirusTotal's code scanner. Here's the controlled experiment that proved it and the fix that cleared it. -->
<!-- slug: clawhub-skill-scan-isolation -->

## Auth-heavy curl examples in SKILL.md trigger VirusTotal's code scanner. Here's the controlled experiment that proved it.

I published an [OpenClaw](https://openclaw.ai/) skill to [ClawHub](https://clawhub.com/) and it came back `SUSPICIOUS`. The warning pointed at [VirusTotal](https://www.virustotal.com/) Code Insight. The skill was intentionally published, worked fine locally, and contained nothing malicious.

The temptation was to just `--force` it. Instead, I decided to figure out exactly what was triggering the flag.

---

### What wasn't the cause

Before testing anything, I listed the plausible causes:

1. Stale or provocative wording in the skill description
2. Missing version metadata
3. Any mention of API keys or auth headers
4. Network mutation examples (POST/PUT/DELETE)
5. Only the top-level `SKILL.md` being scanned aggressively
6. The overall shape of the skill bundle being "too risky"

Most debugging stops here — someone picks the most likely explanation, changes something, and if the warning goes away, declares victory. But that doesn't tell you *which* variable actually mattered.

---

### The experiment

ClawHub enforces a rate limit of 5 new skill slugs per hour. So the experiment used five throwaway slugs at `0.0.1`, then bumped versions on those same slugs for subsequent probes. All probes were deleted after results were recorded.

#### Batch 1: Isolating the trigger class

| Probe | Content | Result |
|---|---|---|
| `baseline@0.0.1` | Plain markdown, no network examples | `CLEAN` |
| `get@0.0.1` | Public unauthenticated `GET` curl example | `CLEAN` |
| `auth@0.0.1` | Authenticated `GET` curl example | **`SUSPICIOUS`** |
| `post@0.0.1` | Authenticated `POST` curl example | **`SUSPICIOUS`** |
| `min@0.0.1` | One minimal authenticated write example | **`SUSPICIOUS`** |

First signal: public read-only examples were fine. A single authenticated external command was enough to trigger the flag.

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

### The root cause

ClawHub's remote security scan treats `SKILL.md` as the most sensitive surface. That makes sense — it's the file agents read first and follow most directly. When that file contains executable command examples with authentication headers, the scanner flags it as potentially dangerous.

The scanner isn't wrong, exactly. A skill that instructs an agent to run authenticated HTTP mutations *is* a higher-risk pattern than one that just describes a workflow. The issue was that my skill had detailed API examples (curl commands with `Authorization: Bearer` headers, POST bodies, endpoint paths) right in the top-level instructions.

This is still an empirical conclusion, not a reverse-engineered VirusTotal rule. ClawHub doesn't expose the precise signature. But nine controlled probes with consistent results is enough to act on.

---

### The fix

Restructure the skill to separate "what to do" from "how to call the API":

- **`SKILL.md`**: High-level, workflow-oriented. Describes *what* the agent should accomplish, *when* to use the skill, and *what* auth is required — in prose, not in executable examples.
- **`references/agent-api.md`**: Exact headers, endpoints, request bodies, response shapes. All the operational detail an agent needs, but in a reference file rather than the top-level instructions.

Published the restructured skill as `v2.0.3`. Result: `Security: CLEAN`.

---

### The operating rules

For anyone publishing OpenClaw skills to ClawHub:

1. Keep `SKILL.md` high-level and workflow-based
2. Move raw curl examples, auth headers, and mutation-heavy examples into `references/`
3. Describe auth requirements in prose at the top level — don't embed executable examples
4. Never normalize `--force` when ClawHub says a skill is suspicious — figure out *why* first
5. If a future skill gets flagged, repeat the probe-based isolation method instead of guessing

---

### Why this matters beyond ClawHub

The broader pattern here is [security scanning for LLM tool use](https://arxiv.org/abs/2403.04957). As agent ecosystems grow, the tools agents can invoke become an attack surface. A skill that tells an agent "run this curl command with this auth token" is functionally the same as giving an agent a shell script. Scanners are right to be cautious about that.

The fix isn't to stop writing skills that do authenticated API calls. The fix is to structure them so the *intent* is clear at the top level and the *implementation* is in reference material. Agents can still access everything they need. Scanners can see that the top-level instructions are descriptive, not imperative.

It's a small structural change, but it's the difference between `SUSPICIOUS` and `CLEAN`.
