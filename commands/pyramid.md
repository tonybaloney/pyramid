---
name: pyramid
description: Run a task through the Pyramid Mixture-of-Agents router — cheap tiers first, escalate only when verification confidence is low.
---

# /pyramid — Pyramid MoA dispatcher

Usage:

```
/pyramid <task description> [--track=claude|gpt|mixed] [--theta=0.75,0.85] [--max-tier=1|2|3] [--anchor=on|off] [--budget=<usd>] [--dry-run=on|off]
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
- `--budget` — optional USD cap on estimated spend. When the projected cost of
  the next escalation would exceed remaining budget, accept the best-so-far
  answer instead.
- `--dry-run` — when `on`, print the DAG and planned tiers but do not dispatch
  any work. Default `off`.

## What this command does

When invoked, you (the host agent) MUST:

1. Parse the user's task description and any flags above. Apply defaults for
   anything not provided. Store the resolved configuration as `pyramid_config`
   (track → tier-model map, theta, max_tier, anchor, budget, dry_run).

2. Invoke the **`pyramid-orchestrate`** skill, passing `pyramid_config` and the
   user's task description. That skill encodes the full Hansen-Zilberstein
   anytime-monitoring control loop.

3. After orchestration completes, present:
   - The aggregated final answer / code edits.
   - The **cost ledger** produced by `pyramid-aggregate`, showing per-subtask
     tier reached, escalation reason, and estimated tokens / cost.

Do **not** attempt to solve the task yourself — your role for this command is
strictly to drive the pyramid skills.
