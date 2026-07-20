# Hermes Prewalk

Frontier planning + executor handoff for [Hermes Agent](https://github.com/NousResearch/hermes-agent). Based on [Stencil's prewalk research](https://stencil.so/blog/prewalk).

## What It Does

Prewalk hands off the **context window**, not a plan document:

1. **Phase 1 (frontier model)** — explores the codebase, creates a todo list, completes exactly one task (edit + verification), then pauses with a structured `⏸️ PAUSE` checkpoint
2. **Phase 2 (human)** — reviews the plan + first task, optionally compresses context, switches models via `/model <executor>`, says "continue"
3. **Phase 3 (executor)** — inherits all exploration context, works through remaining todos strictly in order, one at a time, verifying each before moving on

### Why Not /plan?

Traditional `/plan` makes the frontier model read everything at frontier prices, write a 2K-token "postcard", then the cheap model re-reads everything to execute. You pay for reads twice. Benchmarks show `/plan` costs **14% more** than no handoff at all.

Prewalk's one completed task proves the approach survived contact with the code. The executor imitates a *verified* example, not a plan fragment.

## Quick Start

### 1. Install the Skill

```bash
# From this repo
hermes skills install https://raw.githubusercontent.com/westfable/hermes-prewalk/main/prewalk/SKILL.md

# Or add as a tap source for future updates
hermes skills tap add https://github.com/westfable/hermes-prewalk
hermes skills install prewalk
```

### 2. Apply the Hermes Source Patch (enables `/prewalk` slash command)

```bash
cd ~/path/to/hermes-agent
git apply /path/to/hermes-prewalk/hermes-agent-prewalk.patch
```

Or manually apply the changes from the patch file (419 lines, 4 files).

### 3. Use It

```
/prewalk Build a settings page with tabbed sections
/pw Fix the noise block in the hero section on desktop
/prewalk --into cline-pass/deepseek-v4-flash Build a dashboard widget
/prewalk --max-todos 8 Redesign the archive section with drill-down
```

Flags:
- `--no-pause` — auto-swap to default executor (`cline-pass/deepseek-v4-flash`) after frontier turn
- `--into MODEL` — auto-swap to named executor (implies no pause)
- `--max-todos N` — override the default 12-item todo cap

## Skill Structure

```
prewalk/
├── SKILL.md                              # Workflow + pitfalls (main document)
├── references/
│   ├── concept.md                        # Research, benchmarks, cheating-reduction
│   ├── prompt-frontier.md                # Full Phase 1 instruction
│   ├── prompt-executor.md                # Full Phase 3 continuation prompt
│   ├── model-selection.md                # Frontier/executor pairing guidance
│   ├── task-selection.md                 # 6-point rubric for picking prewalk tasks
│   ├── plan-design-review.md             # Design review patterns, Framer Motion gotchas
│   ├── frontend-verification.md          # Playwright screenshots + DOM geometry probes
│   ├── css-animation-performance.md      # Scroll jank diagnosis
│   └── implementation.md                 # Hermes internals (registry, handler, auto-swap)
└── templates/
    └── task-template.md                  # Task description template + examples
```

## Requirements

- [Hermes Agent](https://github.com/NousResearch/hermes-agent) (any recent version)
- For `/prewalk` slash command: the source patch applied (see step 2)
- For manual mode: any executor model — local LLM via LM Studio/Ollama, or API

## Benchmarks (from Stencil's research)

| Arm | Cost | Duration | Pass |
|-----|------|----------|------|
| Opus 4.8 oneshot | $2.78 | 10.1 min | 84.6% |
| Opus 4.8 + /prewalk | $1.46 | 402s | 78% (+30% vs Flash oneshot) |
| GPT-5.6 Sol oneshot | $1.71 | 372s | 88% |
| GPT-5.6 + /prewalk | $1.04 | 300s | 85% (+10% vs Luna oneshot) |

Prewalk achieves **92–97% of frontier performance at 53–61% of the cost**, with the added benefit of **3× less cheating** (web-searching for known fixes).

## License

MIT — see [LICENSE](LICENSE)

## Credit

Based on ["You only need the frontier model for one single edit"](https://stencil.so/blog/prewalk) by Can Bölük, Stencil (2026-07-13). Skill architecture and Hermes implementation by [@westfable](https://github.com/westfable).
