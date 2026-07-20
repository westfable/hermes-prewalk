---
name: prewalk
description: "Use when building web apps or design tasks via /prewalk command. Frontier model explores, plans, completes exactly one task, then pauses for model switch. Executor continues from todo list."
version: 2.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [prewalk, planning, model-switching, web-app, design, cost-optimization]
    homepage: https://stencil.so/blog/prewalk
    related_skills: [plan, todo]
---

# Prewalk: Frontier Planning + Executor Handoff

Hand off the **context window**, not a plan document. The frontier model explores, creates a todo list, completes exactly one task (edit + verification), then pauses. The executor inherits all the exploration context, the todo list as a steering mechanism, and one completed, verified task as an in-context example.

Based on [Stencil's prewalk research](https://stencil.so/blog/prewalk): the expensive part of an agent's work is **reading** (91% of tokens), not editing (9%). Traditional `/plan` makes the frontier model read everything at frontier prices, write a 2K-token "postcard", then the cheap model re-reads everything to execute — you pay for reads twice, and benchmarks show `/plan` costs *more* than no handoff at all. Full research, benchmarks, and the cheating-reduction effect: `references/concept.md`.

## When to Use

- Triggered by the `/prewalk` slash command (or alias `/pw`)
- Web app building: new pages, features, components, API routes, design system work
- Design tasks: UI layouts, styling, component composition, responsive design
- Tasks complex enough to benefit from frontier planning but where execution can be delegated

**Don't use for:**
- Simple one-off questions or single-file fixes (just ask directly)
- Tasks that require web search during planning (prewalk forbids it)
- Tasks so simple that the frontier model would finish in one task anyway
- **Very complicated, cross-cutting tasks.** The article is explicit: if the task is genuinely complicated, a single planning pass isn't the answer — let a frontier model explore, then dispatch sub-agents (`delegate_task`) per workstream. Prewalk is for *contained* scope; a game of telephone doesn't help with sprawl.

**Note on invocation shape:** The skill triggers when the user requests a web app or design task complex enough to warrant planning, regardless of whether they typed `/prewalk` literally. The slash command is a convenience wrapper, not a requirement. If a user says "let's redesign X" and X fits the criteria above, follow this workflow naturally.

## The Three-Phase Workflow

```
Phase 1 (frontier)      Phase 2 (human)         Phase 3 (executor)
explore → plan →        review → /compress? →   check off PAUSE →
complete task #1 →      /model <executor> →     execute tasks 2..N →
add ⏸️ PAUSE → stop     "continue"              verify each → report
```

### Phase 1: Frontier Exploration + One Task (frontier model)

The frontier model receives a structured prewalk instruction (full prompt: `references/prompt-frontier.md`):

1. **Explore** the codebase thoroughly — config files, components, pages, layouts, routes, API routes, data models. Grep for patterns. Build deep context. Everything read here is context the executor never re-reads.
2. **Analyze the design system** — colors, typography, spacing tokens, component library, existing conventions. Mandatory for web/design tasks.
3. **Visual bug reports: reproduce before planning** — if the task describes a seen defect (noise block, misalignment, glitch), do the visual pass *before* writing todos: `browser_navigate` to the page, `browser_vision` screenshot, `browser_console` DOM probes (getBoundingClientRect + getComputedStyle z-index/position of the section and its decorative layers) to identify the culprit element. Todos then name the exact class/element instead of "investigate X".
4. **Create a todo list** using the `todo` tool — max 12 items (overridable with `--max-todos N`). Each item is a complete task: concrete file path + what to do + verification criterion. Item #1 is the foundational task everything else builds on.
5. **Complete task #1 — and only task #1.** Make its edit(s), run its verification, mark it done. This proves the approach survived contact with the code and gives the executor a *verified* in-context example of the expected pattern, style, and verification cadence.
6. **Add a ⏸️ PAUSE todo** as the current in-progress item and stop.
7. **No web search** during planning.

**Turn budget:** the frontier phase should land in ~7–10 turns (article median: ~7). This isn't just cost discipline — stopping the frontier model early, while it's still in its confident phase, is what suppresses the desperation behaviors (web-searching for known fixes, thrashing). If the phase balloons past ~12 turns with no todo list and no completed task, interrupt: the task is probably too big for prewalk, or the model is lost.

### Phase 2: Human Review + Model Switch (user)

The ⏸️ PAUSE todo is the structured checkpoint. Review:
- [ ] Todo list covers the full scope; every item has a verification criterion
- [ ] Task #1 is genuinely complete: edit made, verification ran and passed, item checked off
- [ ] Task #1 matches codebase conventions (the executor will copy it verbatim)
- [ ] Approach makes sense for the codebase

**Manual mode** (default): Run `/model <whatever-you-want>`. Typically a local LLM via LM Studio (Gemma 4 26B, Qwen 3.6 36B, etc.), Ollama, or any other backend. The model choice is entirely up to you — prewalk just pauses and lets you pick.

**Auto mode** (`--no-pause` or `--into <model>`): The system automatically swaps to the executor after the frontier turn completes — `--into` names it, `--no-pause` alone uses `cline-pass/deepseek-v4-flash`. No human checkpoint.

**Context compression**: If the exploration phase was long (many files read), consider running `/compress` before switching to a model with a smaller context window. Only when the conversation is visibly long.

### Phase 3: Execution (executor model, same session)

**Manual mode**: After switching via `/model`, type `continue` (or your own continuation prompt). **Auto mode**: sent automatically. Full continuation prompt: `references/prompt-executor.md`.

The executor inherits:
- Full exploration context (all reads, greps, dead ends — the "understanding")
- The todo list (steering mechanism — it cannot forget what to do)
- One completed, verified task (in-context example of pattern + verification cadence)
- The ⏸️ PAUSE item as current todo (checks it off and continues)

The executor context is the *opposite* of desperation: the approach already survived contact with the code, so the imitation machine executes instead of flailing.

## Slash Command

```
/prewalk <task description> [--no-pause] [--into MODEL] [--max-todos N]
```

- **`<task description>`** — everything after the command, treated as the task
- **`--no-pause`** — auto-swap to the default executor (`cline-pass/deepseek-v4-flash`) after the frontier turn. Skips the human checkpoint
- **`--into MODEL`** — auto-swap to a *named* executor (mirrors omp's `--prewalk-into`). Implies no pause
- **`--max-todos N`** — override the default 12-item todo cap (use sparingly — too many items loses the executor)
- Alias: `/pw`

Hermes-internals wiring (registry, handler, auto-swap, context-pruning divergence from the article): `references/implementation.md`.

## Model Selection

| Mode | Frontier | Executor | How to switch |
|------|----------|----------|---------------|
| Manual (default) | `cline-pass/glm-5.2` | User's choice (local LLM via LM Studio, Ollama, etc.) | `/model <name>` |
| Auto (`--no-pause`) | `cline-pass/glm-5.2` | `cline-pass/deepseek-v4-flash` | Automatic |
| Auto (`--into M`) | `cline-pass/glm-5.2` | `M` | Automatic |

Local-executor considerations (context windows, quality floor, when to fall back to frontier) and the full cline-pass model list: `references/model-selection.md`.

## Selecting a Good Prewalk Task

Score candidates on six criteria — exploration surface (10+ files), contained scope (8–12 todos), existing conventions to copy, a natural foundational task #1, no blocking data dependencies, clear verification per todo. Full rubric, scouting procedure for existing codebases, and a worked example: `references/task-selection.md`. Task-description template with examples: `templates/task-template.md`.

## Common Pitfalls

1. **Todo items too vague** — "build the auth page" gives the executor nothing concrete. Each item needs a file path and a verification criterion: "Create `path/to/file` with [specifics]. verify: [checkable criterion]"

2. **Task #1 too big** — a multi-file feature as the first task defeats the purpose. Task #1 should be one file, one concept, verified. If it can't be, restructure the todo list until item #1 is genuinely foundational and self-contained.

3. **Frontier model doesn't stop after task #1** — the prompt says one task, but sometimes the model keeps going into task #2. Interrupt with Esc or `/stop`. The whole point is to stop it in its confident phase.

4. **Frontier model stops without verifying** — an unverified task #1 is a plan fragment, not a completed task. The executor will copy the sloppiness. Task #1 counts only when its verification criterion ran and passed.

5. **Too many todo items** — 30 items means the executor gets lost (the article notes GPT-5.6 loves 60-item lists and batch-completes them). Cap at 12; group related work into single items with sub-steps.

6. **Executor batch-completes or declares done prematurely** — the documented small-model failure mode. The executor prompt forbids skipping ahead and requires re-reading the todo list before declaring completion. If it persists, the model is below the floor for this task — switch back to frontier: `/model cline-pass/glm-5.2`

7. **Context window overflow** — exploration reads too many files, executor model can't hold it all. Run `/compress` before switching models.

8. **Model cheats (web search)** — the frontier prompt forbids web search during planning. If the model searches anyway, interrupt and re-run.

9. **Design system mismatch** — task #1 must match existing conventions exactly, or the executor copies the wrong pattern. Read 2-3 existing components before making the edit.

10. **Treating post-handoff refinements as new prewalks** — when the user comes back with "improve X" or "fix Y on mobile" after the handoff, handle the refinement inline: add a todo, do the work, verify, done. Only restart prewalk if the refinement needs fresh exploration. Do NOT re-issue the PAUSE checkpoint for iterations. See "Iteration After Initial Handoff" below.

11. **Over-verifying visual-only changes** — for CSS-only or pure-styling refinements, `npx tsc --noEmit` is sufficient. Skip `npm run build` unless the change touches logic, routes, or imports; unrelated pre-existing route errors (e.g. an `/account/mfa` prerender failure) surface as false failures.

12. **Skipping visual reproduction for visual bug reports** — when the task describes a *seen* defect, do NOT plan from code alone. Reproduce it visually first (screenshot + DOM geometry probes), then write todos that name the exact element/class. In a ROGUE FM hero pass, four overlapping decorative layers were candidates and only geometry probing could attribute the artifact.

13. **Narrow default viewport hides desktop regressions** — the built-in browser screenshot renders at a narrow viewport by default; "desktop looks fine" shots miss ≥1024px layout bugs. Verify responsive/desktop CSS with Playwright pinned to explicit desktop dimensions (e.g. 1600×900) and probe ambiguous pixels with `elementsFromPoint`. Both recipes: `references/frontend-verification.md`.

## Iteration After Initial Handoff

Users iterate after the first pass ("make it responsive", "thicker progress bar", "add a pulse"). How to handle:

- **Small refinement (single file, 1–3 CSS/component changes):** Add a todo inline, do the work, `npx tsc --noEmit` + lint, report. No re-pause, no re-explore.
- **Medium refinement (2–3 files, new visual concept):** Add 2–3 todos inline. Safe even with a local executor — the todo list caps the work. Read any new files first, then todo it.
- **Large refinement (new feature branch, multi-component change):** Ask whether the user wants a new prewalk. If yes, re-run; if no, plan inline with the full todo structure.

The ⏸️ PAUSE is for the initial handoff only, never for iterations.

## Verification Checklist

**Phase 1 exit criteria (frontier):**
- [ ] Codebase explored (files read, patterns grepped, design system understood)
- [ ] Todo list created with ≤ 12 complete tasks, each with file path + action + verification criterion
- [ ] Task #1 completed: edit made, verification ran and passed, item checked off
- [ ] No edits beyond task #1's scope; no web search performed
- [ ] ⏸️ PAUSE todo added and in progress; frontier stopped
- [ ] Frontier phase stayed within ~7–10 turns

**Phase 2/3 (handoff + execution):**
- [ ] User reviewed plan and task #1 before switching (manual mode)
- [ ] Model switched via `/model` (manual) or automatically (`--no-pause`/`--into`)
- [ ] Executor checked off PAUSE and works items strictly in order, one at a time
- [ ] Every item's verification runs before the next item starts
- [ ] Completion declared only with zero unchecked items; skipped items reported

**Per-edit verification cadence:**
- After every edit: `npx tsc --noEmit` (~5–10s)
- After edits touching logic/imports: `npm run lint`
- After edits touching routes/server components: `npm run build` (expect unrelated pre-existing prerender errors on some routes)
- CSS-only polish: `tsc` is enough; defer full build to commit time

## Reference Files

| File | Contents |
|---|---|
| `references/concept.md` | Research, benchmarks, turn-budget rationale, cheating-reduction data (condensed from the Stencil article) |
| `references/prompt-frontier.md` | Full Phase 1 instruction the slash command constructs |
| `references/prompt-executor.md` | Full Phase 3 continuation prompt |
| `references/model-selection.md` | Frontier/executor pairing guidance + full cline-pass model list |
| `references/task-selection.md` | 6-point rubric + scouting procedure for picking prewalk tasks |
| `references/plan-design-review.md` | Plan-mode design review: patterns to challenge before saving, Framer Motion gotchas, patch discipline |
| `references/frontend-verification.md` | Playwright desktop-pinned screenshots + DOM geometry-probe recipes |
| `references/css-animation-performance.md` | Diagnosing scroll jank: enumerate animated layers, classify property cost |
| `references/implementation.md` | Hermes slash-command internals: registry, handler, auto-swap, `--into`, context-pruning divergence |
| `templates/task-template.md` | Template + examples for structuring a prewalk task description |
