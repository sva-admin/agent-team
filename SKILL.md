---
name: agent-team
description: Assemble a cost-correct multi-agent team for a coding or research goal. Assigns each role the cheapest Claude model that can actually do it, pins the model so subagents never inherit an expensive one, and gates fan-out behind a three-question test so you don't pay a 5x token tax for work that was never parallel. Trigger on "build an agent team", "multi-agent", "agent swarm", "orchestrate agents", "which model should I use", "subagents for", "split this across agents", "my Claude Code bill is too high".
---

# Agent Team

Most multi-agent advice tells you to spawn more agents. Measured data says
that's usually how you triple your bill and get slower.

This skill does three things:

1. Routes each **role** to the cheapest model that can do it.
2. Pins `model:` in every agent file so nothing silently inherits Opus.
3. Gates fan-out behind a test, because fan-out is a cost multiplier by default.

## Step 1: Name the roles, not the tasks

Don't ask "how many agents?" Ask "what distinct jobs exist here?" There are
only four, and most goals need two or three of them.

| Role | Job | Model | Why |
|---|---|---|---|
| **Scout** | Find things. Read, grep, glob, locate, summarize. Never writes. | `haiku` | Search is pattern-matching, not judgment. Paying Opus rates to run `grep` is the single most common overspend. |
| **Builder** | Write code to a spec that already exists. Mechanical edits, migrations, tests, boilerplate. | `sonnet` | Sonnet is strong at focused execution. If the spec is clear, a bigger model adds cost, not correctness. |
| **Planner** | Decompose an ambiguous goal. Architecture, tradeoffs, sequencing. Writes the spec the Builder follows. | `opus` | Ambiguity is where model tier actually shows up. |
| **Adversary** | Try to break what was just built. Find the bug, the edge case, the wrong assumption. | `opus`, or `fable` for the last pass before shipping | Adversarial review is the highest-value expensive call you can make. It's cheaper than the incident. |

Assign one agent per role. Don't create `frontend-builder`,
`backend-builder`, and `test-builder`. That's three copies of the same
role paying three setup costs. One Builder, three sequential tasks.

## Step 2: Pin the model in every agent file

This is the step everyone skips and it's the expensive one.

Since Claude Code 2.1.198 (1 July 2026) subagents **inherit the session
model**. The built-in `Explore` agent used to always run Haiku; it now
inherits, capped at Opus. If you're driving an Opus session, your file-search
agent is an Opus agent.

Every agent file gets an explicit `model:`. No exceptions, no `inherit`.

```markdown
---
name: scout
description: Read-only codebase search. Locates files, symbols, and call sites. Never edits.
model: haiku
tools: Read, Grep, Glob
---
```

To take back the built-in Explore agent, define your own with the same name
and it overrides:

```markdown
---
name: Explore
description: Fast read-only search across the codebase.
model: haiku
tools: Read, Grep, Glob
---
```

Ready-made files for all four roles are in `agents/`. Copy them into
`.claude/agents/` and they work as-is.

## Step 3: Gate the fan-out

Before you run N agents in parallel, all three must be true:

1. **Genuinely independent.** No shared state, no ordering constraint, no
   agent needs another's output.
2. **Big enough units.** Each unit is more than a handful of tool calls.
   Below that, spawn overhead dominates the work.
3. **You want the answer, not the reasoning.** Subagent context is discarded;
   only the summary comes back. If you need to see the deliberation, do it
   inline.

Any "no" means run it sequentially. Sequential is cheaper and, measured, often
faster.

**The numbers, so this isn't a vibe.** Instrumented fan-out against a
sequential baseline on the same task:

| Setup | Metered tokens | Wall clock |
|---|---|---|
| Sequential (Opus) | 1x | 4m 15s |
| 2 subagents (Opus) | 2.6x (~5x price-weighted) | 8m 00s |
| 5 subagents (Opus) | 3.2x | 4m 45s |
| Sequential (Fable) | 1x | 2m 15s |
| 2 subagents (Fable) | 5.9x | 7m 00s |
| 5 subagents (Fable) | 4.7x | 17m 00s |

Fan-out never won on time in that study. It's a tool for genuinely parallel
surfaces, not a default.

## Step 4: Route escalations by failure mode, not by task size

When an agent gets it wrong, there are two different dials and picking the
wrong one wastes money.

| What you observed | Dial | Move |
|---|---|---|
| It had all the context, clearly tried, still wrong | **model** | Go up one tier. This is a knowledge gap. |
| It skipped files, didn't run tests, didn't check its work | **effort** | Raise effort, same model. This is a thoroughness gap. |
| Task is mechanical and it did fine | **model** | Go *down* a tier. You're overpaying. |
| Long, many-step, holds a lot of state | **model** | Fable pulls furthest ahead here. |

Raising effort on a knowledge gap makes a confident wrong answer more
thorough. Upgrading the model on a thoroughness gap pays more for the same
skipped test.

## Step 5: Verify, then report cost

End every team run with the Adversary, not with the Builder. A team that
reports "done" without an adversarial pass hasn't verified anything, it has
just finished.

Then state what the run cost in model terms: which roles ran, on which
models, how many calls. If most calls weren't Scout and Builder calls, the
team is mis-routed.

## Assembling a team: worked example

Goal: "Add rate limiting to our API and make sure it doesn't break auth."

Wrong shape (five agents, all inheriting Opus):
`api-agent`, `auth-agent`, `test-agent`, `docs-agent`, `review-agent`,
fanned out in parallel. Five copies of the same expensive model, all
racing on files that touch each other.

Right shape:

1. **Scout** (`haiku`): find every route definition, the auth middleware, and
   existing rate-limit code if any. Returns paths and line numbers.
2. **Planner** (`opus`): read the Scout report, decide where the limiter
   belongs relative to auth, write the spec including the failure mode when
   both trip.
3. **Builder** (`sonnet`): implement the spec. Sequential, one file at a time,
   because auth and routing touch each other.
4. **Adversary** (`opus`): try to get an unauthenticated request past the
   limiter, and a limited request past auth. Report what broke.

Four calls. One expensive planning call, one expensive review call, and the
volume work on cheap models. No fan-out, because step 3 fails the
independence test.

## Reference

- `reference/routing-table.md` - the full model-per-role table with current prices
- `reference/cost-math.md` - how to estimate a team run before you make it
- `agents/` - four ready-to-copy agent files with `model:` pinned

## Anti-patterns

- Creating an agent per file or per feature. Roles, not tasks.
- Leaving `model:` off an agent file. It will inherit the expensive one.
- Fanning out to "go faster" without checking independence. Measured, it's
  often slower.
- Skipping the Adversary because the Builder said it was done.
- Using Fable for volume work. Fable is for the hardest single call in the
  run, not the most calls.
