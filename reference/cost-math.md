# Estimating a team run before you make it

You don't need precision. You need to know whether you're about to spend
cents or dollars, and whether fan-out is worth it.

## The back-of-envelope

For each role, estimate: **number of calls x output tokens per call x output
price**. Output dominates; input is usually a rounding error once prompt
caching is on (caching cuts input cost by up to 90%, and the Batch API is a
flat 50% off if latency doesn't matter).

Worked example, the rate-limiter task from the skill:

| Role | Model | Calls | Output tok/call | Total output | Cost |
|---|---|---|---|---|---|
| Scout | Haiku 4.5 | 3 | 1.5K | 4.5K | $0.02 |
| Planner | Opus 5 | 1 | 3K | 3K | $0.08 |
| Builder | Sonnet 5 | 2 | 6K | 12K | $0.18 |
| Adversary | Opus 5 | 1 | 4K | 4K | $0.10 |
| **Total** | | **7** | | **23.5K** | **~$0.38** |

Now the same task run the wrong way: five feature-named agents all inheriting
an Opus session, fanned out.

| Role | Model | Calls | Output tok/call | Total output | Cost |
|---|---|---|---|---|---|
| 5 agents | Opus 5 | 5 | 6K | 30K | $0.75 |
| plus fan-out overhead | Opus 5 | - | 2.6x multiplier | 78K | $1.95 |

Same task, roughly 5x the bill, and measured slower. That's the entire
argument for this skill.

## The fan-out multiplier, measured

From an instrumented study with a hash-chained audit trail, fan-out against a
sequential baseline on identical tasks:

| Model | 2 subagents | 5 subagents |
|---|---|---|
| Sonnet | 4.2x tokens | - |
| Opus | 2.6x tokens (~5x price-weighted) | 3.2x tokens |
| Fable 5 | 5.9x tokens | 4.7x tokens |

Wall clock, same study:

| Model | Sequential | 2 subagents | 5 subagents |
|---|---|---|---|
| Opus | 4m 15s | 8m 00s | 4m 45s |
| Fable 5 | 2m 15s | 7m 00s | 17m 00s |

Fan-out didn't win on time in a single configuration tested. Treat "parallel
is faster" as a claim to verify on your own workload, not a given.

## When fan-out does pay

The three-question gate from the skill, restated as the economics:

1. **Genuinely independent work.** Shared state means agents redo each other's
   reads. That's where the multiplier comes from.
2. **Units large enough to amortize setup.** Each subagent pays a context
   setup cost. If the unit of work is smaller than that cost, fan-out is pure
   overhead.
3. **You only want the answer.** Subagent context is discarded and only the
   summary returns. If you need the reasoning, you will re-derive it, paying
   twice.

The clean case: N independent files each needing the same mechanical
transformation, where no file imports another. That genuinely parallelizes.
"Add a feature that touches routing, auth, and tests" doesn't.

## Instrumenting for real

Estimates get you to a decision. Measurement gets you a policy.

- `agentcost-cli` (PyPI) attributes token cost per session across Claude Code,
  Cursor, and Codex
- `claudecosts.com` for a hosted view
- `claudexor` for quota-aware routing across agents
- Claude Code's own `/cost` for the current session

Instrument first, then optimize. Optimizing a routing policy you haven't
measured is how people end up with elaborate configs that save nothing.
