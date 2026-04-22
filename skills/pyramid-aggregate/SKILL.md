---
name: pyramid-aggregate
description: Merge Pyramid MoA subtask results into a single user-facing response, write a per-run report file, and print a summary at the configured verbosity.
---

# pyramid-aggregate

Final step of the Pyramid MoA pipeline. Combines per-subtask results into a
single coherent answer, **writes a Markdown per-run report** to disk, and
prints a terminal summary at one of three verbosity levels.

## Inputs

- `results`: map of subtask id → `pyramid-result` block
- `ledger`: list of `{subtask_id, tier, verifier_confidence, decision, reason, cost_used}`
- `dag`: original `pyramid-dag` block
- `task`: original user task description
- `trace`: structured trace dict from `pyramid-orchestrate` (with `subtasks`,
  `totals`, `config`, etc.)
- `pyramid_config`: includes `output_level` (`quiet | normal | debug`,
  default `quiet`) and `session_dir`

## Step 1 — Always: write the per-run report file

**Pre-condition (B2 fix, v0.2.1):** the JSON trace was written by
`pyramid-orchestrate` *before* this skill was invoked. Verify that
`<session_dir>/files/pyramid-runs/<task-slug>-<YYYYMMDD-HHMMSS>.json`
exists. If it does not, write it now from `trace` before doing anything
else, and append a `⚠ JSON trace was written late by pyramid-aggregate`
warning to the always-on warnings list. Do not proceed past this step
until both the JSON trace and (later in this step) the Markdown report
are on disk.

Render the full report as Markdown to:

```
<session_dir>/files/pyramid-runs/<task-slug>-<YYYYMMDD-HHMMSS>.md
```

Using the same `task-slug-timestamp` stem as the JSON trace from
`pyramid-orchestrate` so the two files pair up.

Report contents (in order):

1. **H1**: the original task.
2. **Config**: track, theta, max_tier, anchor, budget, output_level,
   value_of_correctness — as a fenced code block.
3. **DAG**: render `dag.nodes` as an indented tree (parent → children),
   not raw JSON. Show `id (kind, prior=X.XX)`.
4. **Per-subtask sections**: for each node in topological order:
   - Tier path (e.g., `T1 → T2 → accept`).
   - Final verifier confidence and any `issues` from the last verdict.
   - The full `answer` text (or a `…[truncated at 32 KB]` marker if larger).
5. **Cost ledger** (the table from Step 2 below).
6. **Savings summary** (from Step 3 below).
7. **Appendix — raw blocks**: for debugging, dump each subtask's raw
   `pyramid-result`, `pyramid-verdict`, and `pyramid-decision` blocks.

The full report file is written **regardless of `output_level`** — it is
the durable artifact users can open after the fact.

## Step 2 — Format the cost ledger

Render as a markdown table:

```
| Subtask           | Final tier | Confidence | Decision path        | Tokens (in/out) | Cost (tw) |
|-------------------|------------|------------|----------------------|-----------------|-----------|
| <id (kind)>       | T1/T2/T3   | 0.XX       | T1→accept            | 1234 / 567      | 612.3     |
| ...               | ...        | ...        | ...                  | ...             | ...       |
| **Total**         |            |            |                      | I / O           | X.X       |
```

`Cost (tw)` is in token-weighted units per `pyramid-verify`'s model.
`Decision path` is reconstructed from `subtask.tier_path` plus the final
accept/max-tier outcome.

## Step 3 — Compute the savings summary

```
always_tier3_cost = (totals.tokens_in_total + totals.tokens_out_total)
                    * multiplier(track_models[3])
savings_pct       = (always_tier3_cost - totals.cost_tw_units) / always_tier3_cost * 100
```

One-liner:

```
Pyramid MoA: cost = X.X tw-units; always-tier-3 baseline = Y.Y → saved Z%.
```

Note any subtasks that hit `max_tier` without `accept`, prefixed with `⚠`.

## Step 4 — Print the terminal summary at the right verbosity

| `output_level` | Terminal output                                                                |
|----------------|--------------------------------------------------------------------------------|
| `quiet` (default) | Two lines: a tick + cost summary, plus a `Full report:` path. **No answer body, no DAG, no tables.** (C3 fix, v0.2.1.) |
| `normal`       | Quiet output **plus** the synthesized final answer **plus** the cost-ledger table. |
| `debug`        | Normal output **plus** the rendered DAG tree **plus** every raw `pyramid-*` block (today's behavior). |

### Hard contract — quiet means quiet (added in v0.2.1, fixes C3)

When `output_level == "quiet"`, the terminal output **must** be exactly:

1. Zero or more `⚠` warning lines (one per warning).
2. The two-line `quiet` template below.

The host agent must NOT also append the synthesized answer body,
markdown headers, bullet lists, tables, or any other commentary at
quiet level. The full answer is in the report file written in Step 1
— that is the user's escape hatch. If the user wants the answer inline,
they pass `--output=normal`.

### `quiet` output template

```
✓ Pyramid MoA done — <N> subtasks, cost <X> tw-units (vs <Y> always-T3 → <Z>% saved).
  Full report: <absolute path to .md report>
```

### Always-on warnings (any verbosity)

Even at `quiet`, surface a one-line `⚠` warning above the summary for:

- Any subtask that hit `max_tier` without `decision == "accept"`.
- Any malformed solver output (handled per `pyramid-orchestrate` failure rules).
- Any `cost_estimated: true` calls (so the user knows token counts were
  approximated, not measured).

## Step 5 — Sanity checks

Before returning:

- Every subtask in `dag.nodes` must appear in both `results` and the ledger.
  Missing → `⚠ Subtask <id> did not complete` line above the summary.
- If the response mentions file paths, `glob` them and warn in the report
  file (and as a `⚠` line at quiet level) if any do not exist.
- Cap the report file at 1 MB; truncate any individual `answer` field
  beyond 32 KB with `…[truncated]`.

## Output

Print only the summary appropriate to `output_level`. The full detail
lives in the report file written in Step 1; users open it on demand.
