---
title: "Why Your Claude Code Orchestrator Silently Stops Dispatching Subagents"
date: 2026-07-21T01:11:21+03:00
draft: false
slug: "claude-code-orchestrator-silently-stops-dispatching-subagents"
description: "My orchestrator stopped dispatching subagents and never threw an error. The cause was where the file lived. The rule that caused it has since changed, but the failure mode hasn't."
tags: ["claude-code", "ai-agents", "llm", "multi-agent"]
cover:
    image: "posts/images/multi-agents.png"
    alt: "Dispatching subagents silently fails"
---

The first version of [Suhail](https://github.com/wessamfathi/suhail), my Claude Code orchestrator, had a bug that never threw an error. The orchestrator was supposed to dispatch five role subagents per unit of work: researcher, planner, coder, reviewer, auditor. Instead, runs would "complete" with the orchestrator doing everything itself, inline, in one context window. No dispatch ever happened, and nothing in the transcript flagged it.

The root cause was a placement decision that looked obviously correct: the orchestrator is an agent, so I defined it in `agents/`.

## What actually happened

In Claude Code, a file under `agents/` becomes a subagent, invoked through the Agent tool. At the time (this was May 2026), a subagent did not receive the Agent tool itself: subagents could not spawn subagents. So when `/su` invoked my orchestrator, Claude Code handed it a system prompt full of dispatch instructions and a toolset that couldn't dispatch anything.

What makes this failure mode dangerous is that a model missing a tool doesn't crash. It improvises. My orchestrator read "dispatch the researcher subagent," found no Agent tool, and resolved the contradiction the way models resolve contradictions, by doing the closest thing it could. Sometimes it did the research itself inline. Sometimes the session quietly fell back to driving the pipeline from the top level. Either way the run produced output, which is exactly why I didn't catch it immediately. The whole postmortem is public in the project's [decision log](https://github.com/wessamfathi/suhail/blob/main/docs/decisions.md).

## The fix

I moved the orchestrator body out of `agents/` and into `commands/` as a slash command. A slash command's body is injected into the top-level session, and the top-level session has the Agent tool. Same prompt, different placement, and dispatching worked. The five role agents stayed in `agents/` where they belong: single-dispatch, single-artifact workers.

That gave me a rule of thumb I used for every role since: orchestrators and interviewers are slash commands; `agents/` is only for workers that do one job and return one artifact.

## What's changed since (verified July 2026)

Half of that rule has expired. As of Claude Code v2.1.172, subagents can spawn their own subagents, up to five levels deep. If I hit this bug today, the orchestrator-in-`agents/` design would mostly work. Anthropic has also merged custom commands into skills, though `commands/*.md` files keep working as before.

But "mostly" is doing real work in that sentence, because the underlying failure mode, a missing tool producing improvisation instead of an error, is still there. Three ways your dispatcher can silently lose the Agent tool today:

1. **A `tools` frontmatter list that omits `Agent`.** Omitting the `tools` field entirely inherits all tools, including Agent. But the moment you write a `tools` list to restrict a subagent, it becomes an allowlist, and forgetting `Agent` on an orchestrator role removes dispatch with no warning.

2. **The depth limit.** A subagent at depth five doesn't receive the Agent tool at all. The limit is fixed and not configurable. Deep delegation chains hit the same silent wall my v0.1.0 orchestrator did, just five levels later.

3. **Session-state tools never reach subagents.** `AskUserQuestion`, `EnterPlanMode`, and a few others depend on the top-level session and aren't available to subagents even if you list them in `tools`. So the interviewer half of my rule still stands: a multi-turn interview role has to live in a slash command, because a subagent dispatch is one-shot and can't hold a conversation with the user.

## How to catch it

The symptom to watch for: your transcript says "dispatching the researcher" but the work appears inline in the same context, and the subagent panel shows no tree under your orchestrator. Claude Code's panel shows descendant counts per agent, so a dispatcher that's actually dispatching is visible at a glance.

The durable fix is to design for the failure mode instead of the specific constraint: assume any dispatch can silently not happen, and verify outputs instead of trusting narration. Suhail checks after every dispatch that the expected artifact file exists and contains its required sections, and writes a blocker instead of advancing if it doesn't. That gate has caught every variant of this problem since, including ones I didn't predict.

## The takeaway

When an agent is missing a tool, you don't get an error. You get a model doing its best impression of the tool, and a pipeline that looks like it's working. Whatever orchestration constraint you're relying on today (nesting rules, tool inheritance, depth limits) will change under you, so don't encode the constraint, encode the check: verify that every dispatch actually produced its artifact.

---

*The postmortem here is from v0.1.0 of [Suhail](https://github.com/wessamfathi/suhail), the orchestrator I use daily against production Expo/Supabase repos. Claude Code behavior verified against the live docs and changelog as of July 2026 (v2.1.172 changed the nesting rule).*

