<div align="center">

<img src="https://raw.githubusercontent.com/sva-admin/claude-skills/main/assets/sv-academy-512.png" width="96" alt="SV Academy">

# agent-team

**Your file-search agent is running on Opus. Here is the fix.**

A Claude Code skill that routes each agent role to the cheapest model that can
actually do the job, pins it so nothing inherits, and gates fan-out behind a
test instead of a vibe.

[Install](#install-30-seconds) · [The routing table](#the-routing-table) · [Why fan-out isn't free](#why-fan-out-is-not-free) · [SV Academy](https://sv-academy.org)

</div>

---

## The problem

Since Claude Code 2.1.198 (1 July 2026), **subagents inherit the session
model**. The built-in `Explore` agent used to always run Haiku. It now
inherits, capped at Opus.

So if you drive an Opus session, your `grep` runs at Opus prices. Your test
runner runs at Opus prices. A fan-out that used to spin up cheap explorers now
spins up copies of the most expensive model you have.

Most people haven't noticed because the bill is one number.

## Easiest install: just ask

Paste this to Claude Code, Codex, or Cursor:

> **Install the agent-team skill from github.com/sva-admin/agent-team**

Your agent reads this file and does the rest. Nothing to type into a terminal.

<details>
<summary>If you would rather run it yourself</summary>

## Install (30 seconds)

This is an [Agent Skills](https://agentpatterns.ai/standards/agent-skills-standard/)
skill. The canonical location is `.agents/skills/`, which Codex, Cursor, VS Code
and ~40 other agents read directly:

```bash
mkdir -p .agents/skills/agent-team
curl -sL https://github.com/sva-admin/agent-team/archive/main.tar.gz | tar xz --strip-components=1 -C .agents/skills/agent-team
```

Claude Code reads `.claude/skills/`, so symlink it. This is what the ecosystem
already does, no duplication:

```bash
mkdir -p .claude/skills .claude/agents
ln -sfn ../../.agents/skills/agent-team .claude/skills/agent-team
cp .agents/skills/agent-team/agents/*.md .claude/agents/
```

No git required, and nothing but the skill lands in your project.

</details>

### What is portable here, and what is not

Being straight about it, because "just move the folder" would be misleading:

| | Portable? |
|---|---|
| `SKILL.md` and everything in `reference/` | **Yes.** Plain markdown, any compliant agent reads it. |
| The break-even rule | **Yes, fully provider-agnostic.** See below. It works for any tiered pricing. |
| `agents/*.md` frontmatter | **Partly.** `name` and `description` are standard; `model:` and `tools:` are runtime extensions. Compliant agents ignore unknown keys rather than failing. |
| The price table and the inheritance bug | **No.** Those are Anthropic and Claude Code specifics. That is the point of the skill, not an oversight. |

## The routing table

Four roles. Not four agents per feature, four roles total.

| Role | Job | Model | Why |
|---|---|---|---|
| **Scout** | Find things. Read, grep, glob, locate. Never writes. | `haiku` | Search is pattern matching, not judgment. |
| **Builder** | Execute a spec that already exists. | `sonnet` | Strongest per dollar on focused execution. |
| **Planner** | Decompose an ambiguous goal into that spec. | `opus` | Ambiguity is where tier actually shows up. |
| **Adversary** | Try to break what was just built. | `opus` / `fable` | Cheaper than the incident. |

Prices per million tokens, 27 July 2026: Haiku 4.5 `$1/$5` · Sonnet 5 `$3/$15`
(intro `$2/$10` through 31 Aug) · Opus 5 `$5/$25` · Fable 5 `$10/$50`.

**The diagnostic:** if your Opus and Fable output tokens outweigh your Haiku
and Sonnet output tokens, the team is mis-routed. That's the only metric that
matters.

## Escalate by failure mode, not by task size

Two dials, and people pull the wrong one constantly.

| What you observed | Dial | Move |
|---|---|---|
| Had full context, clearly tried, still wrong | **model** | up one tier (knowledge gap) |
| Skipped files, didn't run tests | **effort** | raise effort, same model (thoroughness gap) |
| Mechanical task, done fine | **model** | down one tier, you're overpaying |
| Long, many-step, lots of state | **model** | Fable pulls furthest ahead |

Raising effort on a knowledge gap buys you a more thorough wrong answer.
Upgrading the model on a thoroughness gap pays more for the same skipped test.

## The rule, without the brand names

This part has nothing to do with Claude. Any provider with tiered pricing gives
you the same decision, so here it is in general form.

Trying a cheaper model first is a bet. If it succeeds you keep the difference.
If it fails you paid for both. So:

```
price ratio        = cheap output price / expensive output price
break-even         = hit rate must exceed the price ratio
fraction you save  = hit rate  -  price ratio
```

**That last line is the whole thing.** The savings and the break-even are the
same equation: you save exactly `hit rate minus price ratio`, which is positive
precisely when the bet is worth taking. Nothing else to compute.

Fill in your own provider:

| | Your numbers |
|---|---|
| Cheap model, output price | `$______ / Mtok` |
| Expensive model, output price | `$______ / Mtok` |
| Price ratio (cheap / expensive) | `______ %` |
| Your cheap model's hit rate | `______ %` |
| **Fraction you save** | **hit rate minus price ratio** |

Worked with Claude's published prices, July 2026. Sonnet is `$15/Mtok` out and
Fable is `$50/Mtok` out, so the price ratio is 30%. Sonnet scored 77% solo on
SWE-bench Pro; if your hit rate on your own work is similar:

```
77%  -  30%  =  you pay 47% less than always using Fable
```

The same arithmetic kills a bad idea too. Sonnet backed by **Opus** is a price
ratio of 60% (`15/25`), so at a 77% hit rate you save only 17%, and below a 60%
hit rate you lose money. Plain English version of all of it:

> **Only try the cheap model first when the backup costs a lot more. If the
> backup is only slightly more expensive, just start with the better one.**

One caveat that matters: this assumes you can actually detect the cheap model's
failures. If bad output slips through unchecked, you are not saving money, you
are shipping bugs. That is what `agents/adversary.md` is for.

## Why fan-out isn't free

Everyone posts the diagram where one smart model fans out to N cheap workers
and everything gets faster. Here is what happens when someone instruments it
against a sequential baseline on the same task:

| Setup | Metered tokens | Wall clock |
|---|---|---|
| Sequential (Opus) | 1x | **4m 15s** |
| 2 subagents (Opus) | 2.6x (~5x price-weighted) | 8m 00s |
| 5 subagents (Opus) | 3.2x | 4m 45s |
| Sequential (Fable 5) | 1x | **2m 15s** |
| 2 subagents (Fable 5) | 5.9x | 7m 00s |
| 5 subagents (Fable 5) | 4.7x | 17m 00s |

Fan-out didn't win on time in a single configuration tested. It cost 2.6x to
5.9x the tokens.

Fan-out is a real tool. It's a tool for **genuinely independent work on a
large surface**, not a default posture. So this skill gates it:

1. **Genuinely independent?** No shared state, no ordering, no agent needs
   another's output.
2. **Units big enough?** Each unit is more than a handful of tool calls, or
   spawn overhead eats the gain.
3. **Only want the answer?** Subagent context is discarded. Need the
   reasoning? Do it inline or pay twice.

Any "no" means sequential. Sequential is cheaper and often faster.

## What is in here

```
agent-team/
├── SKILL.md                    the skill Claude loads
├── agents/
│   ├── scout.md                haiku    · read-only search
│   ├── builder.md              sonnet   · executes a spec
│   ├── planner.md              opus     · writes the spec
│   └── adversary.md            opus     · tries to break it
└── reference/
    ├── routing-table.md        full table, prices, escalation rules
    └── cost-math.md            estimate a run before you make it
```

Every agent file has `model:` pinned. That's the point.

## Worked example

Goal: *add rate limiting to our API without breaking auth.*

**The shape most people build:** `api-agent`, `auth-agent`, `test-agent`,
`docs-agent`, `review-agent`, fanned out in parallel. Five copies of Opus
racing on files that import each other. Roughly 5x the bill, measured slower,
and two of them will conflict.

**The shape this skill builds:**

1. **Scout** (`haiku`) finds every route definition, the auth middleware, and
   any existing limiter. Returns paths and line numbers.
2. **Planner** (`opus`) decides the limiter goes *before* auth, so an
   unauthenticated flood is rejected before it costs a DB lookup. Names the
   failure mode: if both trip, the 429 must win or you leak token validity.
3. **Builder** (`sonnet`) implements it. Sequential, because auth and routing
   touch the same files.
4. **Adversary** (`opus`) tries to get an unauthenticated request past the
   limiter and a limited request past auth. Reports what broke.

Seven calls, about $0.38. Two expensive calls, both worth it. The volume work
on cheap models. No fan-out, because step 3 fails the independence test.

## Measure it yourself

Don't take our table on faith, and don't take the fan-out diagram on faith
either. Instrument your own workload:

- `/cost` in Claude Code for the current session
- [`agentcost-cli`](https://pypi.org/project/agentcost-cli/) for per-session
  attribution across Claude Code, Cursor, and Codex
- [`claudexor`](https://github.com/razzant/claudexor) for quota-aware routing
- [`awesome-agent-orchestrators`](https://github.com/andyrewlee/awesome-agent-orchestrators)
  if you want the wider landscape

## Sources

The numbers here aren't ours. Grounded in the last 30 days:

- Anthropic on [choosing a model and effort level](https://claude.com/blog/claude-model-and-effort-level-in-claude-code)
- Systima's instrumented [subagent fan-out study](https://systima.ai/blog/subagent-tax) (SHA-256 hash-chained audit trail)
- Tembo on the [2.1.198 Explore-agent inheritance change](https://www.tembo.io/blog/claude-code-subagents)
- [r/ClaudeCode](https://www.reddit.com/r/ClaudeCode/) on subagent inheritance and role-per-model routing

---

<div align="center">

**Built by [SV Academy](https://sv-academy.org)**, we teach kids to design,
build, and ship real AI apps, live on the internet.

If this saved you money, the follow is the thank-you.

[Website](https://sv-academy.org) · [Instagram](https://www.instagram.com/siliconvalley.academy) · [TikTok](https://www.tiktok.com/@siliconvalley.academy) · [YouTube](https://www.youtube.com/@SVAcademyTH) · [Facebook](https://www.facebook.com/profile.php?id=61590784208590) · [LINE](https://lin.ee/5mwmz71)

MIT licensed. Take it, fork it, ship it.

</div>
