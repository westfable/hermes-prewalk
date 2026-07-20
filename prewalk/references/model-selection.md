# Model Selection for Prewalk

## Mode Summary

| Mode | Command | Frontier | Executor | How to switch |
|------|---------|----------|----------|---------------|
| Manual (default) | `/prewalk <task>` | `cline-pass/glm-5.2` | User's choice | `/model <name>` |
| Auto | `/prewalk --no-pause <task>` | `cline-pass/glm-5.2` | `cline-pass/deepseek-v4-flash` | Automatic |

## Manual Mode: Choosing Your Executor

In manual mode, prewalk pauses after the first completed task (edit + verification). You switch to whatever model you want via `/model`. Common choices:

### Local LLMs (via LM Studio, Ollama, etc.)

- **Gemma 4 26B** — strong general-purpose model, good at following established patterns
- **Qwen 3.6 36B** — excellent code generation, strong instruction following
- Any model loaded in LM Studio or Ollama that's accessible via OpenAI-compatible API

**When to use a local executor:**
- Zero per-token cost (just electricity/compute)
- Privacy — code never leaves your machine
- Good for iterative work where you can supervise

**Context window considerations:**
- Local models may have smaller effective context than API models
- If exploration was thorough (50-100K+ tokens), run `/compress` before switching
- Signs the executor is struggling: repeatedly re-reading files, losing track of the todo list, hallucinating paths — switch back to frontier if this happens

### API Executors (via clinepass)

- **`cline-pass/deepseek-v4-flash`** — cheapest API option, fast, good instruction following
- **`cline-pass/deepseek-v4-pro`** — stronger than flash, still cheaper than frontier
- **`cline-pass/qwen3.7-plus`** — alternative budget option

## Auto Mode: The Default API Pair

`--no-pause` uses `cline-pass/glm-5.2` as frontier and auto-swaps to `cline-pass/deepseek-v4-flash` as executor. This is the fully automated prewalk from the Stencil blog — no human checkpoint.

## Full cline-pass Model List

All models available via the `clinepass` provider (`base_url: https://api.cline.bot/api/v1`, `key_env: CLINE_API_KEY`):

| Model | Typical Role | Notes |
|-------|-------------|-------|
| `cline-pass/glm-5.2` | Frontier (default) | Best reasoning in the list |
| `cline-pass/kimi-k2.7-code` | Frontier alternative | Code-specialized |
| `cline-pass/kimi-k2.6` | Frontier alternative | Older Kimi version |
| `cline-pass/deepseek-v4-pro` | Frontier/executor | Mid-tier cost |
| `cline-pass/deepseek-v4-flash` | Executor (auto mode) | Cheapest, fast |
| `cline-pass/mimo-v2.5` | Executor alternative | Budget option |
| `cline-pass/mimo-v2.5-pro` | Executor alternative | Mid-tier |
| `cline-pass/minimax-m3` | Executor alternative | |
| `cline-pass/qwen3.7-max` | Frontier alternative | |
| `cline-pass/qwen3.7-plus` | Executor alternative | |

## Choosing a Pair

### For cost optimization
- Frontier: `cline-pass/glm-5.2` (does the expensive reading)
- Executor: local LLM (zero per-token cost) or `cline-pass/deepseek-v4-flash` (cheapest API)

### For quality maximization
- Frontier: `cline-pass/glm-5.2` or `cline-pass/kimi-k2.7-code`
- Executor: `cline-pass/deepseek-v4-pro` (stronger than flash, still cheaper than frontier)

### For speed
- Auto mode with `cline-pass/deepseek-v4-flash` as executor (fastest completion in benchmarks)

## Switching Models

To switch to the executor after the frontier pauses:
```
/model <executor-model-name>
```

The `/model` command preserves the full conversation history — the executor sees everything the frontier did. This is the key mechanism that makes prewalk work: the "hand off the context window" mechanic.

For `--no-pause` mode, the slash command handler auto-switches using the same model-swap mechanism as `/moa` (save current model config, swap, run executor turn).

## Signs the Executor Is Struggling

- Repeatedly re-reading files the frontier already read
- Losing track of the todo list
- Hallucinating file paths or component names
- Declaring the task done prematurely

If this happens, switch back to the frontier model for remaining items:
```
/model cline-pass/glm-5.2
```
