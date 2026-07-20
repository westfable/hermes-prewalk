# Prewalk Concept: Why Hand Off Context, Not Plans

Condensed from "You only need the frontier model for one single edit" by Can Bölük,
Stencil, 2026-07-13 (https://stencil.so/blog/prewalk).

## The Problem with /plan

The `/plan` approach: expensive model reads code, thinks deeply, writes a plan document. A model 1/10th the price executes the plan. "Senior architect, junior engineer."

**It costs MORE, not less.** Benchmark data (SWE-bench Pro):

| Arm | Cost | Duration | Pass |
|-----|------|----------|------|
| Opus 4.8 alone (no handoff) | $2.78 | 10.1 min | 84.6% |
| Opus 4.8 + /plan (Opus plans, Flash executes) | $3.18 | 12.7 min | 84.6% |

The "cost-saving" measure costs 14% more. Why?

## The Bill Is O(reads)

Across 1.81B tokens and ~2M tool calls:
- **9%** of tokens are edits/writes
- **91%** of tokens are reads

This split is not a quirk of one harness. Any agent, any model, any scaffold: the bill is `O(reads)`.

People price agents like people: senior time is expensive, so minimize senior involvement. But the expensive part of an agent's day is not the fixing, building, or even the thinking. **The expensive part is the reading.**

## Why /plan Duplicates the Cost

1. Frontier model reads everything at frontier prices (~100K tokens of grounded context)
2. Plan document is a ~2K-token "postcard" from that context
3. Executor gets the postcard, not the understanding
4. Executor must rebuild the rest at its own expense — re-reads everything the frontier already read

Walk through every reason you'd reach for /plan:

- **"I want the deep understanding of the big model."** The understanding lives in 100K+ tokens of grounded context: files read, dead ends eliminated, hypotheses tested. The plan document is a 2K-token postcard from that context.
- **"The task is very complicated."** Then you don't want the main agent executing at all; a single read-only planning turn isn't the answer. Let it explore, then dispatch sub-agents to do the work. A game of telephone doesn't help.
- **"I'm cost constrained."** Reading is the cost. /plan makes the frontier model read everything at frontier prices, then makes the cheap model read it again. You didn't move the expensive part; you duplicated it.

## The Prewalk Solution

Instead of handing off a plan *document*, hand off the *context window itself*:

1. Start the frontier model with a hidden instruction: "plan deeply, then capture the plan as a todo list, then start."
2. Frontier model explores, writes the plan, initializes the todo list.
3. The moment the first task completes (the point where it was confident enough to act and the approach survived contact with the code), swap to the cheap model.
4. Prune the planning instruction from context (where the harness allows it — see note below).

**Why it works:**
- The cheap model never goes "wait, I thought we were planning" — there's no planning instruction left (or, in harnesses that can't prune, the continuation prompt reframes the session as mid-execution)
- As far as it knows, it explored, created a plan, and confidently started executing
- It already made one valid, verified move (a free in-context example)
- The todo list is the critical steering mechanism — the cheap model can forget the plan, but it can't forget the todo reminder

**Hermes note on pruning:** the original implementation prunes the planning instruction from context after the swap. Hermes's `/model` swap preserves the full conversation, including the prewalk instruction. In practice the continuation prompt ("Planning phase complete. You are now the executor...") reframes the context sufficiently, and the PAUSE todo item marks the boundary. If executor confusion is ever observed ("should I be planning?"), this is the first place to look.

## The Turn Budget

The frontier phase is deliberately short: **median ~7 turns** in the benchmark runs. This matters for two reasons:

1. **Cost** — every frontier turn past the first completed task is money spent on work the executor could do.
2. **Behavior** — capable models cheat (web-search for known fixes) when they get desperate, and desperation starts mid-run once exploration stalls: Sol breaks around turn 14, Opus around turn 12. Prewalk terminates the frontier model at the *beginning* of its effort budget — while it's still deriving an approach and landing a first verified task (the confident phase), well before its googling phase begins.

If the frontier phase is ballooning past ~10–12 turns without a todo list and a first task, that's a smell: the task may be too large for prewalk (split it), or the model is lost (interrupt and rescope).

## Results

**Opus 4.8:**
| Arm | Cost | Duration | Pass |
|-----|------|----------|------|
| Flash oneshot | $1.16 | 360s | 60% |
| /prewalk | $1.46 | 402s | 78% (+30%) |
| Opus oneshot | $2.78 | 606s | 85% |

92% of Opus at 53% of the cost, 1.5× the speed, +18 points over oneshot Flash.

**GPT-5.6:**
| Arm | Cost | Duration | Pass |
|-----|------|----------|------|
| Luna oneshot | $0.60 | 570s | 77% |
| /prewalk | $1.04 | 300s | 85% (+10%) |
| Sol oneshot | $1.71 | 372s | 88% |

97% of Sol's pass rate at 61% of the cost, fastest of the three.

## The Cheating Effect

Every SWE-bench task is a bug fixed years ago in public — the answer is on GitHub. Share of runs that searched the web:

| Model | Mode | Cheating |
|-------|------|----------|
| Opus 4.8 | oneshot | 44% |
| Opus 4.8 | /plan | 72% (+28pts) |
| Opus 4.8 | /prewalk | 13% (−31pts) |
| GPT-5.6 Sol | oneshot | 95% |
| GPT-5.6 Luna | oneshot | 100% |
| GPT-5.6 | /prewalk | 70% (−25pts) |

Cheating is what a capable model does when it gets desperate. Prewalk starves it from both ends:
- Frontier model is stopped early (before its googling phase begins — median ~7 turns)
- Executor inherits the *opposite* of desperation: a context where the approach already survived contact with the code. Repro written, first task landed, checklist ticking. Nothing in that context looks like searching, so the imitation machine doesn't search.

Notably, `/plan` makes cheating WORSE than oneshot (+28pts for Opus): its deliverable — a comprehensive document explaining how the fix should work, without ever testing an edit against the code — is exactly the kind of assignment that breeds desperation.

## How Prewalk Got Here

The author occasionally started easy-to-medium tasks with a frontier model, then switched to a cheaper model after a few turns (so it wouldn't fall into its usual thought loops). Benchmarking iterations:

1. **Swap at fixed turn #4** — obviously bad: sometimes the frontier is still lost at turn four, sometimes it has already finished the whole fix.
2. **Swap after first edit** — better, but small models kept declaring the task done out of nowhere.
3. **Todo list + swap after first task** — ask the model to spell out a plan as a todo list with a validation step for each item, then edit code, then trigger the swap. Gating on any edit alone is no good; the todo list is what provides steering. The tiny model can forget the plan, a validation step, or what it's doing entirely, but it cannot forget the todo reminder that bugs it endlessly.

Funny failure mode: GPT 5.6 as the guide likes creating 60-item TODO lists and completing them in batches. An item limit in the prompt is a must.

## Origin: Prefill

None of this is new — it's the oldest trick: prefill. Start the assistant's turn yourself, and the model continues as if the words were its own. A model that small can't be argued into a format, but it can be tricked into one. You can't hand a frontier model ten prefilled tokens anymore (banned at the inference layer nearly everywhere, starting with Anthropic since Sonnet 4.5), but nothing stops you from handing it ten innocently prefilled *turns*: exploration that already happened, a todo list mid-checkmark.

## Upstream Implementations

Stencil upstreamed prewalk to omp, where it ships as `--prewalk`, `--prewalk-into <model>`, or `/prewalk`. The `--prewalk-into <model>` shape (executor named at invocation time) is a candidate enhancement for the Hermes slash command — see `references/implementation.md`.
