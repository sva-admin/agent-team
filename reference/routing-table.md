# The routing table

Prices are per million tokens (input / output), current as of 27 July 2026.
Check [platform.claude.com/pricing](https://platform.claude.com/docs) before
quoting these anywhere; they move.

| Model | Input | Output | Output cost vs Haiku |
|---|---|---|---|
| Haiku 4.5 | $1 | $5 | 1x |
| Sonnet 5 | $3 | $15 | 3x (intro $2/$10 through 31 Aug 2026) |
| Opus 5 | $5 | $25 | 5x |
| Fable 5 | $10 | $50 | 10x |

The whole game is that **output tokens are where the money goes**, and volume
work generates the most output tokens. So the cheapest model has to do the most
work, and the expensive one has to do the least.

## Role to model

| Role | Model | Runs how often | Share of output tokens |
|---|---|---|---|
| Scout | Haiku 4.5 | Most calls in the run | Should be the largest share |
| Builder | Sonnet 5 | Once per implementation step | Second largest |
| Planner | Opus 5 | Once per goal | Small |
| Adversary | Opus 5, or Fable 5 for the final pass | Once per implementation step | Small |

If your Opus or Fable share of output tokens is bigger than your
Haiku plus Sonnet share, the team is mis-routed. That is the whole diagnostic.

## Escalation by failure mode

Two dials. Model tier is for what Claude **knows**. Effort is for how hard
Claude **tries**. Pulling the wrong one is the most common way people spend
more and get the same answer.

| What you observed | Diagnosis | Dial | Move |
|---|---|---|---|
| Had full context, clearly tried, still wrong | knowledge gap | model | up one tier |
| Skipped files, did not run tests, did not verify | thoroughness gap | effort | raise effort, keep the model |
| Mechanical task done correctly | overpaying | model | down one tier |
| Long, many-step, lots of state to hold | tier-sensitive | model | Fable pulls furthest ahead |
| Verbose and slow on something simple | over-effort | effort | lower effort |

Anthropic's own framing: "If Claude has all the pertinent context and clearly
tried and still got it wrong, that's a signal to pick a larger model."
Raise effort when Claude skipped work, not when it lacked knowledge.

## Cheap-effort-on-expensive-model vs high-effort-on-cheap-model

This is genuinely unsettled and worth testing on your own workload. Practitioners
report Fable at low effort outperforming Opus at default for daily work, which
inverts the naive price ordering. Two things to know:

- The comparison is workload-specific. Test it on your repo, not on a benchmark.
- Cost is not monotonic in tier once effort is variable. A low-effort Fable call
  can be cheaper than a high-effort Opus call on the same task, because effort
  drives output token count.

Measure before you commit to a routing policy. `agentcost-cli` and
`claudecosts.com` both exist for this.

## The one line everyone forgets

```markdown
model: haiku
```

Since Claude Code 2.1.198 (1 July 2026) subagents inherit the session model.
The built-in `Explore` agent, which used to always be Haiku, now inherits and
is capped at Opus. Every agent file needs an explicit `model:` or you are
paying session prices for search.
