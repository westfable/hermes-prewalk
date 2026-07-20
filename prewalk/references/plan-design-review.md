# Plan-Mode Design Review: Challenge Your Own First-Blush Solutions

When you're writing a plan, the moment you reach for "wrap X in a new Z" or "add a new flag" or "rename A and B to share an id" — pause. First-blush solutions to design problems are usually 80% right and 20% category-confused. The 20% is where the user will catch you, and the 20% is what makes the plan hard to execute.

This file collects the patterns that consistently need challenging, the questions to ask, and the discipline required to keep the plan honest.

## The "Challenge" Workflow

After Step 6 (Review the Plan) in the `plan` skill, before saving the file, walk every non-obvious design choice and ask:

1. **Is this the cleanest fit, or the first one I thought of?**
2. **What would a reviewer push back on, and what would I say back?**
3. **If I'm hedging ("this is a known pattern", "we add a safety net", "we wrap to be safe"), am I papering over a category confusion?**

If any answer is "I'm hedging", rewrite the design until the hedge goes away.

**Then** — for the 1-3 choices where you'd genuinely benefit from the user's domain knowledge, ask in chat *before* saving. Phrase as "would you say X is the cleanest fit, or would you pick something else?" — that invites the user to challenge your reasoning, not just rubber-stamp it.

Don't ask about mechanical decisions (file naming, exact CSS values, TDD step ordering). Those belong in the plan, not the chat.

## Common Categories Where First-Blush Solutions Are Wrong

### 1. `AnimatePresence` + shared `layoutId`

**First-blush pattern:** "I have an `AnimatePresence mode="wait"` with two branches. To make a shared-element transition work, I'll put `layoutId="X"` on BOTH branches (or wrap them in an outer `motion.div` that holds the id)."

**Why it's wrong:** Two `motion` elements both claiming `layoutId="X"` simultaneously will conflict during cross-fades. The wrap-in-outer-motion-div version works but conflates two transitions that may be conceptually separate.

**Cleanest fit:** ask "are these two transitions actually the same transition, or are they temporally disjoint?" If they're disjoint (e.g. splash→empty-state on first load, then later user-pick-track→track-artwork), the `layoutId` lives on ONE branch only (the empty state) and the other branch uses key-based `AnimatePresence` alone. They never compete for the id because they never happen at the same time.

**Diagnostic:** "would the user ever see this morph happen while the other transition is in progress?" If no, the transitions are disjoint — give each its own mechanism.

### 2. "Merge two `LayoutGroup`s so a third thing can find them"

**First-blush pattern:** "I need element C to morph into A or B depending on viewport. A and B have separate `LayoutGroup`s, so I'll rename them to share an id so C can find the visible one."

**Why it's almost right but wrong-rationalized:** Yes, share the id. But the *reason* isn't "merge them for C" — it's that `layoutId` resolution requires a shared `LayoutGroup` ancestor. Different reason, same change. Get the rationale right because it tells you the mitigation if it doesn't work.

**Cleanest fit:** rename the two groups to a shared id, AND verify that (a) Framer Motion can host multiple distinct `layoutId`s in one group (it can), and (b) the two groups never coexist on screen at the same layoutId (true if `display: none` or breakpoint-gated).

**Mitigation if it breaks:** fall back to mounting the morphing element inside BOTH `LayoutGroup`s (DOM duplication, hidden via CSS). Ugly but works.

### 3. "Add a new flag to distinguish two cases"

**First-blush pattern:** "I need to know if this is a first-load splash or a route-change splash. I'll add an `isInitial: boolean` prop."

**Why it's often wrong:** Two real distinctions = two real concepts. If the splash and the route loader look different, render different, and behave different, they're different *components*, not the same component with a flag. A flag is right when the distinction is "same component, slightly different shape"; a separate component is right when the distinction is "different lifecycle, different motion, different parent context".

**Diagnostic:** "if I removed this flag, would the component still be usable in one of the two cases?" If yes, it's a real variant — flag. If no, it's a different thing — separate component.

**Common case in this codebase:** `V3EmblemLoader` with `variant: "initial" | "route"` — this IS a real variant (same component, ring/no-ring, logo optional). The flag is right. The plan should articulate *why* the variant lives on the same component (shared CSS, shared image, shared accessibility treatment) — not just drop the prop in.

### 4. "Wrap X and Y in a new Z" when X or Y could just be extended

**First-blush pattern:** "I need A to have feature F that B has. I'll create a wrapper that delegates to A or B."

**Cleaner alternatives, in order of preference:**
- Extend A to support F directly (if A is the only thing that would use F *plus* the F-from-B behavior, just put F on A)
- Extract F into its own component that both A and B use (if F is a self-contained piece of UI)
- Use a hook (if F is state, not UI)

**When the wrapper IS right:** when A and B have incompatible props, incompatible lifecycles, or when the user has explicitly described a "container" mental model.

**Diagnostic:** "what does the wrapper do that A couldn't?" If the answer is "decide which one to render", that's the only legit reason. If the answer is "add feature F", extend A.

### 5. "Suppress X to be safe" / "add Y as a defense"

**First-blush pattern:** "We don't want the dashed ring to show on the initial splash, but the route loader uses it. I'll add a CSS rule that hides the ring inside the splash class as a safety net, in case a future caller forgets to pass the right prop."

**Why it's often a tell:** You've identified a confusion in the API (two callers want different things from the same component) and your fix is to paper over it in CSS. The real fix is to make the API unambiguous — e.g. a `variant` prop that's required, not defaulted, for the splash caller. The CSS belt-and-braces is fine to ADD after the API fix, but it shouldn't be the only fix.

**Diagnostic:** "if I remove the safety net, would the failure mode be a bug or a feature?" If a bug, the API is wrong. If a feature (e.g. we want the splash to be the only thing that doesn't show the ring), the API distinction is right and the safety net is just defense in depth.

## The Patch Discipline Lesson

When editing a plan file (or any dense list/table) in place, **re-read the full block before issuing a single atomic patch** — especially after a `read_file` with `offset`/`limit` returns a "partial view" warning.

**The failure mode:** you issue `patch(old_string=X, new_string=Y)` where `old_string` is a fragment of a longer list. The patch tool finds the fragment, replaces it, but the surrounding items (which you didn't see) become orphaned/duplicated/renumbered-broken. The result requires a second patch to clean up — which the user sees as "you made a mess and cleaned it up."

**The fix:** when the edit is "swap item 3 in a list of 8", don't patch item 3 in isolation. Read the whole list, write the whole list back as one `write_file` or one `patch` with the full block as `old_string`.

**Heuristic:** if the change requires renumbering, reformatting, or touching > 1 item in a list, re-read and rewrite the whole block. The save in round-trips isn't worth the cleanup risk.

## Framer Motion Shared-Element Gotchas (Quick Reference)

When the plan involves `layoutId` / `LayoutGroup` / shared-element transitions:

- **`LayoutGroup` with the same `id` merge.** Framer Motion merges groups across the tree by id. Use this when the morphing element is in a different DOM subtree than the target.
- **Multiple `layoutId`s in one group is fine.** Each `motion` element with a unique `layoutId` resolves independently.
- **`layoutId` resolution requires a shared `LayoutGroup` ancestor.** Without it, the morph doesn't trigger.
- **`AnimatePresence` and `layoutId` interact in known ways:** a `motion` with a `layoutId` exiting inside an `AnimatePresence` will participate in the exit transition AND the shared-element transition. If two branches both have the same `layoutId`, the exit of one and enter of the other will fight.
- **Position: fixed on either side is fine.** `position: fixed` on the source or the target doesn't break the morph — Framer measures rects regardless of positioning.
- **Breakpoint-gated groups work.** If group A is `display: none` and group B is visible, the visible group's `layoutId` is the target. No need to "know" which viewport you're in.

## Related Files

- `plan` skill (protected) — Step 6 review, Step 7 design challenge
- `prewalk` SKILL.md — Phase 1 exploration, especially "design system analysis is mandatory"
- `references/concept.md` — why planning-time discipline matters
