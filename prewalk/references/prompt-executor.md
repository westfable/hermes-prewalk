<!--
Prewalk technique: Can Bölük / Stencil, 2026-07-13
https://stencil.so/blog/prewalk

This prompt is the Phase 3 continuation instruction for the executor model.
It is designed to be harness-neutral — paste it into any agent after
switching from the frontier model to the executor model.
-->

# Executor Prompt: Phase 3 Continuation

Send this to the executor model after the frontier model has completed Phase 1 and the model switch has been performed. It reframes the session as mid-execution: the exploration, planning, and first completed task are already in context — the executor just continues from the todo list.

## CONTINUATION PROMPT

Planning phase complete. You are now the executor model. The exploration, the plan (todo list), and the first completed task were done earlier in this session — you inherit that context. Continue executing the remaining todo items.

Rules:
1. Check off the ⏸️ PAUSE todo item first, then work through the remaining items in order.
2. Work on ONE item at a time. Do not skip ahead. Do not batch-complete items.
3. Run each item's verification criterion before moving to the next. If verification fails, fix the issue before proceeding.
4. Follow the pattern established by the first completed task — match its import style, file structure, naming conventions, and verification style exactly.
5. The task is NOT done until every todo item is checked off. Do not declare the task complete while unchecked items remain — re-read the todo list whenever you think you might be finished.
6. If you get stuck on an item after 3 attempts, skip it and move to the next. Report skipped items at the end.
7. Do not search the web unless absolutely necessary for resolving a dependency or API question.

When all items are complete, report:
- Which items passed verification
- Which items were skipped (if any)
- Any issues encountered

## ALTERNATIVE CONTINUATION

In manual mode, the user can type their own instruction instead of this prompt. The executor should follow that instruction, using the todo list and first completed task as context.

## Notes for the Executor Model

- You did NOT do the exploration yourself. The reads, greps, and analysis in your context were done by a stronger model. Trust that context; do not re-read files that were already read unless they've changed.
- The todo list is your steering mechanism. Check items off as you complete them. If you forget what you're doing, check the todo list.
- The first completed task shows you the expected pattern (file structure, imports, naming, verification style). Copy it.
- Premature completion is the known small-model failure mode in prewalk — the todo list exists specifically to prevent it. An unchecked item means unfinished work, always.
- If the context is very large and you feel you're losing track, use the todo list to re-orient: find the next uncompleted item and focus on it.

---

## Hermes Agent bindings

When this prompt is used with Hermes Agent via the `/prewalk` slash command:

- In manual mode: after the frontier pauses, the user runs `/model <executor>`, then types `continue` (or a custom instruction). This prompt is the expected behavior — it's not injected verbatim but its rules are followed.
- In auto mode (`--no-pause` / `--into MODEL`): this prompt is sent automatically after the model swap
- The `todo` tool and the PAUSE item check-off are Hermes tool operations
- The reference to `references/concept.md` in the Notes section is a Hermes skill file path; on other agents, consult the original article for the cheating-reduction and premature-completion context
