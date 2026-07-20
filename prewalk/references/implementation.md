# /prewalk Slash Command Implementation (Hermes internals)

## Overview

The `/prewalk` command is implemented across four files:

1. `hermes_cli/commands.py` — Command registration in COMMAND_REGISTRY
2. `cli.py` — Handler dispatch in process_command() + auto-swap in main loop + attribute init
3. `hermes_cli/cli_commands_mixin.py` — Handler method `_handle_prewalk_command()`
4. `$HERMES_HOME/skills/software-development/prewalk/SKILL.md` — The skill (prompt templates + workflow)

## 1. Command Registration (commands.py)

Added after the `moa` entry in the Session section of `COMMAND_REGISTRY`:

```python
CommandDef("prewalk", "Frontier model explores + plans + completes first task, then pauses for model switch",
           "Session", aliases=("pw",),
           args_hint="<task> [--no-pause] [--into MODEL] [--max-todos N]"),
```

## 2. Handler Dispatch (cli.py process_command)

Added after the `moa` elif block:

```python
elif canonical == "prewalk":
    self._handle_prewalk_command(cmd_original)
```

## 3. Attribute Initialization (cli.py __init__)

Added near the other `_pending_*` attributes:

```python
# Prewalk --no-pause: when True, swap to executor model and seed
# continuation prompt after the current agent turn completes.
self._pending_prewalk_swap_after_turn = False
self._pending_prewalk_executor_model: str | None = None
self._pending_prewalk_restore_model: dict | None = None
```

## 4. Handler Method (cli_commands_mixin.py)

`_handle_prewalk_command(self, cmd: str)`:

- Parses task description + flags (`--no-pause`, `--into MODEL`, `--max-todos N`)
- Constructs the frontier prompt inline (preamble + task + step-by-step instructions — see `references/prompt-frontier.md`)
- For `--no-pause`: saves current model config, records the executor model, and sets `_pending_prewalk_swap_after_turn = True`
- Sets `self._pending_agent_seed = frontier_prompt` — the seed runs as the next agent turn

The `_pending_agent_seed` mechanism is the key enabler: it's picked up by the main loop as the next user message, same as `/moa` and `/blueprint`.

## 5. Auto-Swap in Main Loop (cli.py)

After the `_pending_moa_disable_after_turn` block:

```python
if getattr(self, "_pending_prewalk_swap_after_turn", False):
    self.model = (self._pending_prewalk_executor_model
                  or "cline-pass/deepseek-v4-flash")
    self.agent = None  # force re-init with new model
    self._pending_agent_seed = (
        "Planning phase complete. You are now the executor model. "
        "Continue executing the remaining todo items..."
    )
    self._pending_prewalk_swap_after_turn = False
```

This swaps the model, nulls the agent (forces re-init with new model config on next turn), and seeds the continuation prompt as the next agent turn.

## `--into MODEL` (executor named at invocation)

The upstream omp implementation ships `--prewalk-into <model>`: the executor is named at invocation time and the swap happens automatically after the first task, with no default-model guessing. The Hermes equivalent:

- `--into <model>` implies auto-swap (like `--no-pause`) but to the named model rather than the hardcoded default
- Store in `_pending_prewalk_executor_model` at parse time; the main-loop swap block above already honors it
- `--no-pause` without `--into` keeps the historical default (`cline-pass/deepseek-v4-flash`)
- `--into` combined with a pause (no `--no-pause`) is also useful: the pause happens, but `/model` autocompletes/preconfirms the recorded executor — a nice-to-have, not yet implemented

## Context Pruning (known divergence from the article)

The original prewalk prunes the planning instruction from the executor's context after the swap, so the cheap model never sees "you are planning" text. Hermes's model swap preserves the full conversation — including the frontier prompt. Mitigations, in order of effort:

1. **(current)** The continuation prompt reframes the session as mid-execution; the PAUSE todo marks the boundary. Works in practice.
2. **(possible)** On swap, rewrite the first user message in the session history to just the task description, dropping the FRONTIER_INSTRUCTIONS block. Requires care with prompt caching (a mid-session history rewrite invalidates the cache anyway — the model change already does).
3. **(overkill)** Fork the session at the swap point with a filtered history.

If executor confusion ("should I be planning?") is observed, implement mitigation 2.

## Key Infrastructure Patterns Used

### _pending_agent_seed

Set `self._pending_agent_seed = <message>` and the main loop picks it up as the next user message. This is how `/moa`, `/blueprint`, `/prompt`, and `/prewalk` work.

### Model swap without context loss

Setting `self.model = <new_model>` + `self.agent = None` changes the model without clearing conversation history. The executor inherits the full context window. This is what makes prewalk possible — the "hand off the context window" mechanic.

### /moa as the pattern

The `/moa` command shows how to:
1. Save current model config (`_pending_moa_restore_model`)
2. Swap model config in-place
3. Set `_pending_agent_seed` to the prompt
4. Restore/swap after the turn completes

The `--no-pause` variant follows this exact pattern. Manual mode doesn't need the swap — the user runs `/model` themselves.

## Manual Mode Flow (default)

```
/prewalk <task>
  → handler builds frontier prompt
  → sets _pending_agent_seed = frontier_prompt
  → prints "Prewalk started, will pause after first task"
  → main loop picks up seed, sends to frontier model
  → frontier model explores, creates todos, completes task #1 (edit + verify + check off), pauses
  → user reviews plan + task #1
  → user runs /model <executor> (e.g., local LM Studio model)
  → user sends "continue" or their own prompt
  → executor inherits context, works through remaining todos
```

## Auto Mode Flow (--no-pause / --into)

```
/prewalk --no-pause <task>          # default executor
/prewalk --into <model> <task>      # named executor
  → handler builds frontier prompt
  → saves current model config, records executor model
  → sets _pending_prewalk_swap_after_turn = True
  → sets _pending_agent_seed = frontier_prompt
  → main loop picks up seed, sends to frontier model
  → frontier model explores, creates todos, completes task #1
  → frontier turn completes
  → main loop sees _pending_prewalk_swap_after_turn = True
  → swaps self.model to the recorded executor
  → nulls self.agent (forces re-init)
  → sets _pending_agent_seed = continuation prompt
  → main loop picks up seed, sends to executor model
  → executor inherits context, works through remaining todos
```

## Testing

After implementing, verify:
1. `/prewalk` with no args shows usage
2. `/prewalk <task>` loads skill and sends frontier prompt
3. `/pw <task>` (alias) works the same
4. `--max-todos 5` is parsed correctly
5. `--no-pause` sets up the auto-swap to the default executor
6. `--into <model>` sets up the auto-swap to the named executor
7. After frontier turn, executor model is auto-swapped and continuation prompt is sent
8. Manual mode: after frontier pauses, `/model <name>` works and executor inherits context
9. Frontier stops after exactly one completed task (task #1 checked, PAUSE in progress, no task-#2 edits)
