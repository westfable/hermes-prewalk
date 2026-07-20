# Prewalk Task Selection — Rubric & Scouting Procedure

How to pick (or brainstorm) a task that actually benefits from prewalk. Distilled from a candidate-scouting session on the ROGUE.FM codebase (July 2026).

## The 6-point rubric

Score each candidate task:

1. **Exploration surface** — 10+ relevant files to read (routes, components, data layer, design system). If the whole task touches 2 files, prewalk's frontier phase is wasted — just do it directly.
2. **Contained scope** — one page/section/feature, 8–12 todos max. Cross-cutting refactors and multi-page features exceed the todo cap and lose the executor.
3. **Existing conventions to copy** — shared components, design tokens, similar pages already exist. The executor imitates the frontier's first completed task; with no patterns to anchor on, output drifts.
4. **Natural foundational task #1** — an obvious single file/component that everything else builds on (e.g. the genre grid for a drill-down, the tab bar for a tabbed layout), with a verification the frontier can run before pausing.
5. **No blocking data dependencies** — the tables, API routes, and data the feature needs must already exist. UI for data that isn't there yet gets blocked mid-execution.
6. **Clear verification** — every todo has a checkable criterion (renders, URL restores state, sort works).

## Scouting procedure for an existing codebase

1. **Confirm the target project first.** If the user names one, go straight there — don't survey every repo on the machine.
2. **Map built vs. stub.** `find app -name "page.tsx" | xargs wc -l` (or the framework equivalent). Low line counts = stubs = candidates.
3. **Mine existing planning docs.** Check `.cursor/plans/`, `.windsurf/plans/`, `*.plan.md`, `DESIGN-*.md`, `TODO.md`, roadmap files, `AGENTS.md` / `CLAUDE.md`. Specced-but-unbuilt features come with acceptance criteria already written — ideal prewalk fodder.
4. **Verify plan-vs-code drift.** Roadmap docs lie: features marked "done" may be stubs, and vice versa. Grep for the feature's identifiers before trusting the doc — state variables (e.g. `homeTab`), table names (e.g. `badges`), component names. Zero hits = not built.
5. **Check data dependencies against migrations/API routes** (rubric point 5) before proposing anything data-driven.
6. Present 2–3 candidates, each with a ready-to-paste `/prewalk` prompt, a todo-count estimate, and a recommendation with reasoning.

## Worked example (ROGUE.FM, July 2026)

Scouted candidates from the project's Windsurf V2 plan (`~/.windsurf/plans/roguefm-v2-update-*.md`):

| Candidate | Scope check | Verdict |
|---|---|---|
| Archive genre-first drill-down | One section (`ArchiveSection.tsx`, 269 lines), existing V3 components to copy, no new data needed | ✅ Recommended — most self-contained |
| Homepage tabbed layout | Existing home components; `homeTab` state grep returned 0 hits (not built) but data layer existed | ✅ Good second choice |
| Profile badges/bio | `badge` grep across `supabase/migrations/` returned 0 hits | ⚠️ Riskier — blocked on missing table |

Winning criteria in practice: self-containment (one directory), verifiable todos (each drill-down level checkable in the browser), and zero missing-data risk.

## Prompt shape that works

The ready-to-paste prompts that scored well followed this shape:

```
/prewalk <verb> the <feature> in <project>. <Level-by-level or section-by-section
breakdown of the expected UX>. <Data source / reuse constraint: which existing
components, hooks, or lib functions to use>. <Hard requirement: shareable URL state,
matching conventions, etc.>
```

Key ingredients: concrete file/route anchors, named existing components to reuse (not reinvent), and 1–2 hard requirements the executor can verify against.
