# pyramid — Pyramid Mixture-of-Agents plugin for Copilot CLI

A Copilot CLI plugin that brings **hierarchical Mixture-of-Agents** routing to
your terminal. Hands a task to the cheapest tier first, verifies the answer,
and only escalates to a more expensive tier when a Hansen-Zilberstein
**Value-of-Computation** rule says it's worth it.

Based on Khaled, *Pyramid MoA: A Probabilistic Framework for Cost-Optimized
Anytime Inference*, [arXiv:2602.19509](https://arxiv.org/abs/2602.19509).

> On MBPP, the router intercepts 81.6% of bugs; on GSM8K/MMLU the system
> nearly matches the 68.1% Oracle baseline while achieving up to 42.9%
> compute savings.

## Install

From GitHub (recommended):

```
copilot plugin install tonybaloney/pyramid
```

From a local clone:

```
git clone https://github.com/tonybaloney/pyramid.git
copilot plugin install ./pyramid
```

Verify:

```
copilot plugin list
```

## Usage

In an interactive Copilot CLI session:

```
/pyramid:pyramid <task description> [flags]
```

Copilot CLI namespaces plugin commands as `/<plugin>:<command>`, so the slash
command for this plugin is `/pyramid:pyramid` (not `/pyramid`).

### Flags

| Flag                      | Default          | Description                                                                |
|---------------------------|------------------|----------------------------------------------------------------------------|
| `--track`                 | `claude`         | Model family: `claude`, `gpt`, or `mixed`.                                 |
| `--theta`                 | `0.75,0.85`      | Confidence thresholds θ₁,θ₂ for escalating from tier 1→2 and 2→3.          |
| `--max-tier`              | `3`              | Hard ceiling on escalation.                                                |
| `--anchor`                | `off`            | If `on`, pass the rejected lower-tier answer to higher tiers as context.   |
| `--budget`                | (none)           | Cost cap in **token-weighted units** (tw-units). Stops escalation when hit. |
| `--dry-run`               | `off`            | Print the DAG and planned tiers without dispatching.                       |
| `--output`                | `quiet`          | Terminal verbosity: `quiet`, `normal`, or `debug`. **New default in 0.2.0**. |
| `--value-of-correctness`  | `8`              | VoC scaling factor (was `10` pre-0.2.0; re-tuned for token-weighted costs).|

### Output verbosity

By default (`--output=quiet`) the terminal shows just two lines:

```
✓ Pyramid MoA done — 3 subtasks, cost 1240 tw-units (vs 18200 always-T3 → 93% saved).
  Full report: ~/.copilot/session-state/<id>/files/pyramid-runs/<task>-<ts>.md
```

The full report — DAG, per-subtask answers, cost ledger, raw `pyramid-*`
blocks — is **always** written to disk at the printed path. Open it on
demand for the scientific detail. Use `--output=normal` to also see the
final answer + ledger inline, or `--output=debug` to mirror v0.1.0's
verbose output.

### Cost units

As of v0.2.0, all cost values (`--budget`, ledger totals, savings) are
**token-weighted units**:

```
cost(call) = (input_tokens + output_tokens) * multiplier(model)
```

| Model              | Multiplier |
|--------------------|------------|
| claude-haiku-4.5   | 0.33       |
| claude-sonnet-4.6  | 1.00       |
| claude-opus-4.7    | 7.00       |

GPT-track multipliers are placeholder — see
`skills/pyramid-verify/SKILL.md`. Pre-0.2.0 the cost flag was documented as
USD but always operated on flat ordinals (`1 / 5 / 25`). The new units are
proportional to actual model spend, so the ratio Haiku:Sonnet:Opus is now
`1 : 3 : 21` instead of the legacy `1 : 5 : 25`.

### Tier mapping

| Tier | `claude`             | `gpt`            | `mixed`              |
|------|----------------------|------------------|----------------------|
| 1    | `claude-haiku-4.5`   | `gpt-5-mini`     | `claude-haiku-4.5`   |
| 2    | `claude-sonnet-4.6`  | `gpt-5.4`        | `gpt-5.4`            |
| 3    | `claude-opus-4.7`    | `gpt-5.3-codex`  | `claude-opus-4.7`    |
| V    | `claude-haiku-4.5`   | `gpt-5-mini`     | `claude-haiku-4.5`   |

### Examples

The examples below are real outputs captured from the v0.2.0 smoke run on
this repository. See [`docs/benchmarks/baseline-metrics.md`](docs/benchmarks/baseline-metrics.md)
for the full results, including discovered defects.

#### 1. Read task (low difficulty, single-shot)

```
$ copilot -p "/pyramid:pyramid List all Markdown files modified in the last 7 days in this repository. Read-only task." --autopilot

✓ Pyramid MoA done — 1 subtask, cost 297.0 tw-units (vs 6300.0 always-T3 → 95.3% saved).
  Full report: ~/.copilot/session-state/<id>/files/pyramid-runs/list-all-markdown-…json

⚠ Token counts estimated (`cost_estimated: true`).
```

DAG: 1 read node, prior 0.10. Routed to tier-1 (Haiku-4.5). Verifier
confidence 0.92 → accept. Wall-clock ~134 s.

#### 2. Transform task (medium, multi-subtask)

```
$ copilot -p "/pyramid:pyramid Find any markdown links in README.md that point at a non-existent path or anchor. Do not edit files; report findings only." --autopilot

✓ Pyramid MoA done — 3 subtasks, cost ~478 tw-units (vs ~10,150 always-T3 → ~95% saved).
  Full report: ~/.copilot/session-state/<id>/files/pyramid-runs/find-broken-readme-links-…md
```

DAG: `extract-links` → `validate-targets` → `report`. All accepted at
tier-1. Wall-clock ~163 s.

#### 3. Synthesize task (medium, design-style)

```
$ copilot -p "/pyramid:pyramid Design how to add a --retry-budget=N flag to /pyramid:pyramid …" --autopilot

✓ Pyramid MoA done — 2 subtasks, cost ~3500 tw-units (vs ~75,000 always-T3 → ~95% saved).
  Full report: ~/.copilot/session-state/<id>/files/pyramid-runs/design-how-to-add-a-retry-budget-n-flag-…md
```

DAG: `survey-flag-and-loop` → `design-retry-budget`. Both accepted at
tier-1 (Haiku-4.5 handled the design with confidence 0.82). The full
Markdown report contains the per-subtask answers, file-by-file change
plan, and a 7-step test plan. Wall-clock ~499 s.

#### What gets written to disk

Every run writes two paired files to `<session>/files/pyramid-runs/`:

```
<task-slug>-<YYYYMMDD-HHMMSS>.json   # machine-readable trace (config, DAG, subtasks, totals)
<task-slug>-<YYYYMMDD-HHMMSS>.md     # human-readable report (DAG, answers, ledger, savings)
```

The JSON trace is the source of truth for benchmarking and post-hoc
analysis. The Markdown report is what you open when you want to see why
the router made a particular choice.

#### Caveats observed during the smoke run

- Token counts are currently estimated (`cost_estimated: true` in every
  trace) — the host `task` tool does not yet surface per-call token
  counts to the orchestrator. Treat absolute tw-unit numbers as
  directional; relative comparisons within a single run are reliable.
- The orchestrator may short-circuit the verifier on trivially atomic
  read tasks (a known deviation from the spec; see baseline-metrics.md
  defect B3).
- Wall-clock for trivial tasks is dominated by orchestrator overhead
  (decompose + verify + aggregate inference), not by the subtask itself.

## How it works

```
/pyramid:pyramid <task>
   │
   ▼
pyramid-orchestrate skill ──────────────────────────────────────────────┐
   │                                                                    │
   │ 1. pyramid-decompose       → DAG of subtasks (id, kind, prior)     │
   │                                                                    │
   │ 2. for each subtask:                                               │
   │      ┌──────────────────────────────────────────────────┐          │
   │      │ tier-N solver (task tool, model=tier-N)          │          │
   │      └──────────────────────────────────────────────────┘          │
   │                          │                                         │
   │                          ▼                                         │
   │      ┌──────────────────────────────────────────────────┐          │
   │      │ pyramid-verify  →  pyramid-verifier (Haiku)      │          │
   │      │  → Hansen-Zilberstein VoC: escalate iff           │          │
   │      │    expected_gain × value > cost_ratio             │          │
   │      └──────────────────────────────────────────────────┘          │
   │                          │                                         │
   │            accept ◀──────┴──────▶ escalate to tier N+1            │
   │                                                                    │
   │ 3. pyramid-aggregate       → final answer + cost ledger            │
   └────────────────────────────────────────────────────────────────────┘
```

## Anchoring (paper §4)

Passing a rejected lower-tier answer to a higher tier sounds helpful but the
paper finds:

- **+19.2pp** accuracy when the lower-tier reasoning is correct.
- **−18.0pp** accuracy when it is incorrect.

Since the only reason we escalated is that we *don't trust* the lower-tier
output, the default `--anchor=off` shows higher tiers only the subtask + the
verifier's critique. Use `--anchor=on` only when you have evidence the lower
tier is usually directionally right.

No MCP server, no hooks, no external runtime — everything piggybacks on the
CLI's built-in `task` tool and its per-call `model` override.

## Citation

```
@article{khaled2026pyramidmoa,
  title  = {Pyramid MoA: A Probabilistic Framework for Cost-Optimized Anytime Inference},
  author = {Khaled, Arindam},
  journal= {arXiv preprint arXiv:2602.19509},
  year   = {2026}
}
```

## License

MIT.
