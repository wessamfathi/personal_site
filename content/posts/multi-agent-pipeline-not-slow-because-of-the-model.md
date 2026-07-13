---
title: "Your Multi Agent Pipeline Not Slow Because of the Model"
date: 2026-07-13T16:18:43+03:00
draft: false
slug: "multi-agent-pipeline-not-slow-because-of-the-model"
description: "I timed my Claude Code multi-agent pipeline at 22 minutes and blamed the model. The real bottleneck was dispatch count. Seven mechanical fixes that cut it, backed by a real Suhail run."
tags: ["claude-code", "ai-agents", "llm", "developer-productivity"]
cover:
    image: "/posts/images/suhail.png"
    alt: "Unity CPU Profiler Window"
---

I run [Suhail](https://github.com/wessamfathi/suhail), my own Claude Code orchestrator, against real repos most days. The first time I timed a full run (an indexer plus a trivial two-part plan), it took 22 minutes. My instinct was to blame the model: swap to a faster tier, shave a few seconds per call, ship it.

That instinct was wrong. I pulled the actual dispatch counts, and the bottleneck wasn't model latency at all. It was how many times I was cold-starting a subagent to re-read context it had already seen.

## The real cost: dispatch count, not tokens-per-second

Each "Part" in that run fired five role dispatches: researcher, planner, reviewer, auditor, and the coder. Four of those five paid a full cold start, fresh context, re-reading the same research bundle, the same plan, the same diff, before doing maybe 30 seconds of actual reasoning. The model was fast. The pipeline was slow because I'd designed it to re-derive the same context five times per unit of work.

Once I started treating dispatch count as the metric to optimize instead of "which model is faster," the fixes were mechanical:

1. **Merge independent sibling roles into one agent.** Researcher and planner don't need separate context windows. They need the same context, used twice. I merged them into a single `scout` role, and reviewer+auditor into a single `verifier`. That's 5 dispatches per Part down to 3, a 40% cut with zero loss of rigor. Each merged role still does both jobs, just without re-paying the cold-start tax in between.

2. **Dispatch truly independent siblings in parallel.** If two roles don't depend on each other's output, fire them in the same message instead of sequentially. Obvious in hindsight; easy to miss when you're writing the orchestrator prompt linearly.

3. **Short-circuit when you can synthesize the answer instead of dispatching for it.** A diff touching only `*.md`, `tests/`, `*.test.*`, or static assets doesn't need a full security-audit dispatch. The orchestrator can synthesize `Verdict: clean` from a path heuristic and skip the subagent entirely.

4. **Make cache-use instructions unambiguous.** This one cost me a debugging session. I had a prompt that said both "skip re-deriving the stack" and "discover stack conventions" in different sections. The model resolved that contradiction by just reading everything anyway. Caching only pays off if the instruction to use it has exactly one interpretation.

5. **Trim each role's input bundle to what it actually consumes.** Don't hand the reviewer the full research doc if it only ever reads the plan and the diff.

6. **Remember the orchestrator prompt itself is re-injected every tick.** A 21KB slash-command preamble isn't a one-time cost. You pay it on every single invocation of that command. Keep it short.

7. **Tier models by role, not uniformly.** The code-writing role needs your top model. A role that's only emitting a verdict string doesn't.

## The caching detail most people get wrong

If you're trying to reason about cost (not just speed) in a Claude Code pipeline, a few facts matter and aren't obvious from the docs:

- Cache reads run at 0.1× base price; cache writes run 1.25× (5-minute TTL) or 2× (1-hour TTL).
- The minimum cacheable prefix is model-dependent, running from 1,024 tokens on older models up to 4,096 on the current Opus and Haiku tiers, so a short orchestrator preamble may not even qualify.
- Cache reads don't count toward your input-tokens-per-minute limit on current models (each request still counts toward the requests-per-minute limit). Once you're running pipelines back-to-back, that matters more than the price discount.
- Anthropic's public docs don't state whether Claude Code's own Agent dispatch applies `cache_control` to subagent system prompts. I couldn't find a documented answer, so I verified it empirically: fire two back-to-back identical dispatches and check `usage.cache_read_input_tokens` on the second one. A markdown-and-shell harness can't set `cache_control` itself, so whether you get the benefit depends entirely on what the host does under the hood.

## The takeaway

If your orchestration pipeline feels slow, don't reach for a faster model first. Count your dispatches per unit of work. Every dispatch is a cold start paying full context-reconstruction cost, regardless of how fast the underlying model responds. Cut dispatches before you touch model tier. In my case that single change (5 to 3 roles per Part) was the largest lever by a wide margin, bigger than everything else on this list combined.

---

*Numbers here are from a real measured run of [Suhail](https://github.com/wessamfathi/suhail), the orchestrator I use daily against production Expo/Supabase repos, not a synthetic benchmark.*
