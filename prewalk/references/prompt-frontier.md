<!--
Prewalk technique: Can Bölük / Stencil, 2026-07-13
https://stencil.so/blog/prewalk

This prompt is the Phase 1 instruction for the frontier model.
It is designed to be harness-neutral — paste it into any agent
that supports a todo/task-list tool and mid-session model switches.
-->

# Frontier Prompt: Phase 1 Prewalk Instruction

Send this to the frontier model to begin a prewalk session. Append your task description after the FRONTIER_INSTRUCTIONS block. The model will explore the codebase, create a todo list, complete exactly one task, then stop at a checkpoint.

## FRONTIER_PREAMBLE

You are beginning a prewalk session. Prewalk means: you explore the codebase, build a plan as a todo list, complete EXACTLY ONE task from that list (edit + verification), then STOP for a human checkpoint before model handoff. The executor model will inherit your context window — all your reads, your todo list, and your one completed task. Everything you read now is context the executor won't have to re-read, and your one completed task is the in-context example it will imitate. Make your exploration count, and make your one task exemplary.

Budget: aim to finish exploration + plan + first task within roughly 7–10 turns. If you cannot land a confident first task in that budget, stop and report why instead of grinding.

## FRONTIER_INSTRUCTIONS

### Step 1: Explore

Read and understand the codebase thoroughly:
- Package.json, tsconfig, config files (tailwind, vite, next, etc.)
- Existing components, pages, layouts, route structure
- API routes, server actions, data models
- Design system: colors, typography, spacing tokens, component library
- Grep for patterns already in use (state management, styling approach, auth setup)
- Check test setup and conventions
- **If the task reports a visual defect** ("noise in the corner", "misaligned X", "weird gap"): reproduce it before planning. Navigate a browser to the page, screenshot the affected region, and probe the DOM to attribute the artifact to a specific element/class. Code reading alone cannot pick the culprit among overlapping decorative layers — do the visual pass first, once.

Do NOT search the web during this phase. All knowledge must come from the codebase itself.

### Step 2: Plan

Create a todo list using your agent's task-list tool with these rules:
- Maximum 12 items (override if needed)
- Each item must be a COMPLETE task: specific file path + what to do + how to verify. A task is something that can be finished and checked off, not a fragment of one.
- Ordered: infrastructure → core functionality → polish → validation
- Each item includes a one-line verification criterion
- **Item #1 must be the foundational task** — the single self-contained piece everything else builds on (one file, one concept), because you will execute it yourself and it becomes the executor's in-context example.

Good todo item format:
```
Create `src/app/settings/page.tsx` with tabbed layout using existing `Tabs` component. verify: page renders at `/settings` without errors
```

Bad todo item format:
```
Build the settings page
```

### Step 3: Complete Task #1 — and ONLY Task #1

Execute the first todo item in full:
1. Make the edit(s) that task #1 describes (typically one file, one concept)
2. Run its verification criterion (e.g. `npx tsc --noEmit`, render check) and confirm it passes
3. Mark task #1 as completed in the todo list

Constraints:
- Task #1 must match existing conventions exactly (import style, file structure, naming) — the executor will copy this pattern for every remaining item
- Do NOT start task #2. Do NOT make edits beyond task #1's scope. One task, fully done, verified, checked off.
- If task #1's verification fails, fix it before stopping — a broken first example poisons the executor. But stay within task #1's scope while fixing.

### Step 4: Pause

Add a final todo item (the structured pause task):
```
⏸️ PAUSE — Frontier handoff checkpoint. Human: review the plan + task #1, optionally compress context if large, then switch to the executor model and say "continue".
```

Set it as the current in-progress item. Then STOP and present:
- Summary of your approach (2-3 sentences)
- The todo list state (task #1 checked off, PAUSE in progress, rest pending)
- What task #1 was, how it was verified, and why it's the foundation

Do NOT continue to the next todo item. The human will review, switch models, and tell the executor to continue.

---

## Hermes Agent bindings

When this prompt is used with Hermes Agent via the `/prewalk` slash command:

- It is constructed inline by `_handle_prewalk_command` in `cli_commands_mixin.py`
- `{TASK_DESCRIPTION}` is everything after the command name and flags
- `--max-todos N` overrides the 12-item default
- The Step 4 pause text includes `/compress` (Hermes context compression command) and `/model <executor>` (Hermes model switch command) as the checkpoint actions
- `browser_navigate`, `browser_vision`, `browser_console`, and the `todo` tool are Hermes tool names
- For auto mode (`--no-pause` / `--into MODEL`): the system auto-swaps models and seeds the executor prompt
