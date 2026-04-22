# AGENTS.md

## Project Overview

Pyramid is a Copilot CLI plugin implementing a Mixture-of-Agents (MoA) router
that dispatches each subtask to the cheapest capable solver tier and escalates
only when verification confidence is low. Escalation is governed by the
Hansen–Zilberstein Value-of-Computation rule. The plugin ships four agents
(three solver tiers + one verifier), four orchestration skills, and a single
`/pyramid:pyramid` slash command.

## Repository Layout

```
pyramid-moa/
├── agents/
│   ├── tier1-fast.agent.md
│   ├── tier2-mid.agent.md
│   ├── tier3-deep.agent.md
│   └── pyramid-verifier.agent.md
├── skills/
│   ├── pyramid-orchestrate/SKILL.md
│   ├── pyramid-decompose/SKILL.md
│   ├── pyramid-verify/SKILL.md
│   └── pyramid-aggregate/SKILL.md
├── commands/
│   └── pyramid.md
├── docs/
│   └── benchmarks/
│       ├── tasks.md            # Standard benchmark suite
│       └── baseline-metrics.md # Most-recent benchmark run + diffs
├── plugin.json                 # Manifest (current: 0.2.0)
├── README.md
├── AGENTS.md                   # This file
├── CHANGELOG.md
└── LICENSE                     # MIT
```

## Installation & Usage

```
copilot plugin install tonybaloney/pyramid
/pyramid:pyramid <task description> [flags]
```

See `README.md` for the full flag reference.

## Build / Test / Lint

This is a **declarative plugin** — Markdown + JSON only. There is no
compilation, package manager, or test runner.

- Validate `plugin.json` with any JSON parser before committing.
- Validate the YAML frontmatter at the top of every `*.agent.md` and
  `SKILL.md`.
- Validate fenced ` ```pyramid-* ` JSON blocks by running a representative
  benchmark task through `/pyramid:pyramid --output=debug` and inspecting
  the report file under `~/.copilot/session-state/<id>/files/pyramid-runs/`.
- The benchmark suite under `docs/benchmarks/tasks.md` is the closest thing
  to a test harness; re-run it after non-trivial changes and update
  `docs/benchmarks/baseline-metrics.md`.

## Conventions

### Frontmatter

Every agent and skill markdown file begins with YAML frontmatter:

```yaml
---
name: <kebab-case-id>
description: <one sentence>
tools: [optional, list]
---
```

### Output contracts (strict JSON, fenced)

| Block               | Producer               | Required fields                                                                 |
|---------------------|------------------------|--------------------------------------------------------------------------------|
| `pyramid-dag`       | `pyramid-decompose`    | `task`, `nodes[]` (`id`, `kind`, `description`, `difficulty_prior`, `dependencies`, `success_criteria`) |
| `pyramid-result`    | tier-1 / tier-2 / tier-3 solvers | `subtask_id`, `answer`, `self_confidence` (0.0–1.0), `blocked_reason` (or null) |
| `pyramid-verdict`   | `pyramid-verifier`     | `subtask_id`, `confidence`, `issues[]`, `would_benefit_from_escalation`, `evidence` |
| `pyramid-decision`  | `pyramid-verify`       | `subtask_id`, `verifier_confidence`, `verifier_issues[]`, `decision`, `next_tier`, `reason`, `estimated_cost_used`, `cost_breakdown` |

These schemas are load-bearing. Solver outputs are **captured by the
orchestrator, not surfaced to the user** — at the default
`--output=quiet`, the user sees only a 2-line summary. Do not weaken the
JSON contracts to make terminal output prettier; render the report file
instead.

### Confidence calibration anchors (verifier)

| Confidence | Meaning |
|------------|---------|
| 0.95 | Verifiably correct (independent check passes) |
| 0.80 | Looks correct, no contradictions found |
| 0.55 | Plausible but one concrete concern (edge case, untested) |
| 0.30 | Likely buggy or partially blocked |
| 0.10 | Clearly wrong or empty |

### Cost units (v0.2.0 token-weighted)

```
cost(call) = (input_tokens + output_tokens) * multiplier(model)
```

| Model              | Multiplier |
|--------------------|------------|
| claude-haiku-4.5   | 0.33       |
| claude-sonnet-4.6  | 1.00       |
| claude-opus-4.7    | 7.00       |

`--budget` and all ledger totals are in token-weighted units (`tw-units`).
When the underlying `task` tool does not surface token counts, fall back
to `len(prompt + answer) / 4` and mark the trace entry `cost_estimated:
true` (a `⚠` line is shown to the user).

### Thresholds, priors, and anchoring

- Default escalation thresholds: **θ₁ = 0.75, θ₂ = 0.85**.
- Default `value_of_correctness`: **8** (re-tuned in 0.2.0; was 10).
- `P_correct(N+1 | N wrong)`: tier1→2 = 0.55, tier2→3 = 0.45.
- Difficulty priors come from the heuristic table in
  `skills/pyramid-decompose/`. Tune per-task; do not change the defaults
  without justification in the PR and a re-benchmark.
- **Anchoring is OFF by default.** Per the paper (§4), passing *incorrect*
  lower-tier reasoning to a higher tier degrades accuracy by up to **−18 pp**.
  Correct lower-tier reasoning improves it by **+19 pp**, but we cannot
  reliably tell them apart at escalation time.

### Per-run trace and report

Every non-dry-run invocation writes two paired files:

- `<session>/files/pyramid-runs/<task-slug>-<YYYYMMDD-HHMMSS>.json` — the
  structured trace (single source of truth for metrics).
- `<session>/files/pyramid-runs/<task-slug>-<YYYYMMDD-HHMMSS>.md` — the
  human-readable report.

The terminal shows only what `--output` selects; the trace and report are
always written.

## Agent Roster

### `tier1-fast` (default model: `claude-haiku-4.5`, multiplier 0.33)

- **Role:** First-pass solver for low-to-moderate difficulty subtasks.
- **Invoked when:** `pick_initial_tier` selects tier 1 (difficulty_prior ≤ 0.70).
- **Must NOT:** expand scope, invent dependencies, attempt architecture-level
  work, or fake high `self_confidence`. Report `self_confidence ≤ 0.30` and a
  `blocked_reason` when out of depth — that is how escalation is triggered.

### `tier2-mid` (default model: `claude-sonnet-4.6`, multiplier 1.00)

- **Role:** Multi-step reasoning, light refactoring, moderate ambiguity.
- **Invoked when:** tier 1 is rejected by `pyramid-verify`, or when
  `difficulty_prior > 0.70` causes the orchestrator to skip tier 1.
- **Must NOT:** redo work already accepted from upstream subtasks; treat
  the verifier critique as the contract, not a suggestion.

### `tier3-deep` (default model: `claude-opus-4.7`, multiplier 7.00)

- **Role:** Final escalation tier — hardest subtasks where lower tiers
  failed verification.
- **Invoked when:** tier 2 is rejected and `max_tier ≥ 3`.
- **Must NOT:** be invoked speculatively. Tier-3 dispatch is expensive
  (~21× tier-1 per token) and only justified when the VoC test in
  `pyramid-verify` passes.

### `pyramid-verifier` (default model: `claude-haiku-4.5`, multiplier 0.33)

- **Role:** Strict-JSON verifier; emits a calibrated `pyramid-verdict` for
  a tier output.
- **Invoked when:** After every tier solver call, by `pyramid-verify`.
- **Must NOT:** attempt to solve the subtask itself, rewrite the answer,
  or call other tiers.

## Skill Roster

| Skill                | Purpose |
|----------------------|---------|
| `pyramid-orchestrate`| Main control loop: decompose → dispatch → verify → escalate → aggregate. Owns the per-run JSON trace. |
| `pyramid-decompose`  | Produce the `pyramid-dag` JSON of subtasks with difficulty priors. |
| `pyramid-verify`     | Wrap the verifier call and apply the Hansen–Zilberstein VoC rule (token-weighted). |
| `pyramid-aggregate`  | Merge per-subtask results, write the Markdown report file, and print the terminal summary at the configured verbosity. |

## Adding a New Agent or Skill

1. Create `agents/<name>.agent.md` or `skills/<name>/SKILL.md` with valid
   YAML frontmatter.
2. If the agent emits structured output, define and document its fenced
   `pyramid-*` block schema alongside the existing ones in this file.
3. Register the file in `plugin.json` if the manifest enumerates agents/skills.
4. Update `README.md` and the relevant roster section in this `AGENTS.md`.
5. Add at least one task that exercises the new agent/skill to
   `docs/benchmarks/tasks.md` and re-run the benchmark.
6. Smoke-test by running a representative task through `/pyramid:pyramid
   --output=debug` and confirming the new agent/skill is invoked and its
   output parses.

## Prompt Style for Tier and Verifier Prompts

- **Imperative voice.** "Return the JSON block." not "You could return…".
- **Inline the schema** every time, even if the agent has seen it before.
  Sub-agents are stateless across invocations.
- **Never instruct a sub-agent to call `/pyramid:pyramid` recursively.** Use
  the underlying skills (`pyramid-decompose`, `pyramid-verify`, etc.) directly.
- **Anchoring discipline:** When passing context on escalation, follow
  `pyramid-orchestrate`'s rule — include the rejected answer only if
  `anchor == true`.
- **Mitigations over heroics:** prefer reporting a low `self_confidence` with
  a clear `blocked_reason` over guessing.

## License

MIT — see `LICENSE`.
