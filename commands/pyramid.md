---
name: pyramid
description: Run a task through the Pyramid Mixture-of-Agents router — cheap tiers first, escalate only when verification confidence is low.
---

# /pyramid — Pyramid MoA dispatcher

Usage:

```
/pyramid <task description> [--track=claude|gpt|mixed]
                            [--theta=0.75,0.85]
                            [--max-tier=1|2|3]
                            [--anchor=on|off]
                            [--budget=<tw-units>]
                            [--dry-run=on|off]
                            [--output=quiet|normal|debug]
                            [--value-of-correctness=<float>]
```

## Flags

- `--track` — model family for the tiers. Default: `claude`.
  - `claude`: tier1=`claude-haiku-4.5`, tier2=`claude-sonnet-4.6`, tier3=`claude-opus-4.7`
  - `gpt`: tier1=`gpt-5-mini`, tier2=`gpt-5.4`, tier3=`gpt-5.3-codex`
  - `mixed`: tier1=`claude-haiku-4.5`, tier2=`gpt-5.4`, tier3=`claude-opus-4.7`
- `--theta` — confidence thresholds for escalation as `θ₁,θ₂`. Default `0.75,0.85`.
  Below θ at tier N triggers consideration of tier N+1 (subject to VoC test).
- `--max-tier` — hard ceiling on escalation. Default `3`.
- `--anchor` — when escalating, whether to pass the rejected lower-tier answer
  to the higher tier as context. Default `off` (paper §4 warns of up to −18pp
  accuracy degradation from anchoring on incorrect reasoning).
- `--budget` — optional cost cap in **token-weighted units** (`tw-units`,
  see `skills/pyramid-verify/SKILL.md` for the formula). When the projected
  cost of the next escalation would exceed remaining budget, accept the
  best-so-far answer instead. **Breaking change in 0.2.0**: the flag was
  previously documented as USD but always operated on ordinal units; it now
  uses token-weighted units explicitly.
- `--dry-run` — when `on`, print the DAG and planned tiers but do not dispatch
  any work. Default `off`.
- `--output` — terminal verbosity. Default `quiet`.
  - `quiet` (default): a two-line summary + path to a per-run Markdown report
    written under `~/.copilot/session-state/<id>/files/pyramid-runs/`.
  - `normal`: quiet output plus the synthesized final answer plus the cost ledger.
  - `debug`: everything in `normal` plus the DAG tree and every raw
    `pyramid-result` / `pyramid-verdict` / `pyramid-decision` block. (This
    was the v0.1.0 default; it is now opt-in.)
  Regardless of level, the full report file is always written to disk.
- `--value-of-correctness` — float multiplier in the VoC escalation rule.
  Default `8` (re-tuned in 0.2.0 from the legacy `10` for the new
  token-weighted cost ratios).

## What this command does

When invoked, you (the host agent) MUST:

1. Parse the user's task description and any flags above. Apply defaults for
   anything not provided. Resolve `session_dir` from the current Copilot CLI
   session-state path. Store the resolved configuration as `pyramid_config`
   (track → tier-model map, theta, max_tier, anchor, budget, dry_run,
   output_level, value_of_correctness, session_dir).

2. Invoke the **`pyramid-orchestrate`** skill, passing `pyramid_config` and the
   user's task description. That skill encodes the full Hansen-Zilberstein
   anytime-monitoring control loop and writes a JSON trace + Markdown report.

3. After orchestration completes, the **`pyramid-aggregate`** skill prints the
   summary at the configured `--output` level. Do NOT additionally print the
   ledger or raw blocks yourself — they are already in the report file.

Do **not** attempt to solve the task yourself — your role for this command is
strictly to drive the pyramid skills.

## Post-conditions (hard contract)

When `--output=quiet` (the default), the terminal output **MUST** be:

1. Zero or more `⚠` warning lines (one per warning).
2. Exactly two lines: the cost summary and the full report path.

You **MUST NOT** also print the synthesized answer body, markdown headers, bullet lists, tables, code blocks, or any other commentary at quiet level.

The full answer is in the per-run Markdown report written by `pyramid-aggregate` in Step 1 — that is the user's escape hatch. If the user wants the answer inline, they MUST pass `--output=normal`.

See `skills/pyramid-aggregate/SKILL.md` § "Hard contract — quiet means quiet" for details.
