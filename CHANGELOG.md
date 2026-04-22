# Changelog

All notable changes to this plugin are documented here.

## 0.2.0 — token-weighted costs and quiet UX

### Breaking

- **Cost model replaced.** All cost accounting (`--budget`, ledger totals,
  savings, `pyramid-decision.estimated_cost_used`) is now in
  **token-weighted units** (`tw-units`):
  `cost(call) = (input_tokens + output_tokens) * multiplier(model)`.
  Multipliers — Haiku 0.33, Sonnet 1.00, Opus 7.00 — replace the legacy flat
  ordinals 1 / 5 / 25. The new Haiku:Sonnet:Opus ratio is `1 : 3 : 21` (was
  `1 : 5 : 25`). GPT-track multipliers are placeholder pending confirmation.
- **`--budget` units changed.** Previously documented as USD but always
  operated on ordinal units; now explicitly tw-units.
- **Default verbosity changed.** `--output=quiet` is now the default; v0.1.0
  behavior is reproducible with `--output=debug`.
- **`pyramid-decision` schema gained `cost_breakdown`.** Consumers parsing
  the block must accept the new field; required fields unchanged.

### Added

- `--output=quiet|normal|debug` flag on `/pyramid:pyramid`.
- `--value-of-correctness=<float>` flag (default re-tuned from `10` to `8`).
- Per-run JSON trace at
  `<session>/files/pyramid-runs/<task-slug>-<YYYYMMDD-HHMMSS>.json`
  capturing tokens, costs, latencies, verdicts, and totals.
- Per-run human-readable Markdown report at the same stem with a `.md`
  extension. Always written, regardless of `--output`.
- `AGENTS.md` documenting conventions, agent roster, and skill roster.
- `CHANGELOG.md` (this file).
- `docs/benchmarks/tasks.md` — standard 12-task benchmark suite.
- `docs/benchmarks/baseline-metrics.md` — most-recent measurements,
  before/after comparisons, and acceptance gates.

### Changed

- Default `value_of_correctness` lowered to `8` to reflect the smaller
  Haiku:Sonnet:Opus cost ratio under the new token-weighted model.
- `pyramid-aggregate` now owns the report-file write; the orchestrator
  owns the trace-file write.
- `⚠` warnings (max-tier-without-accept, malformed solver output,
  estimated token counts) are surfaced even at `--output=quiet`.

### Notes

- No behavior change to anchoring (default remains `off`; paper §4).
- The underlying VoC algorithm and θ defaults are unchanged.

## 0.1.0 — initial release

- Three solver tiers + verifier, four orchestration skills, single slash
  command. Ordinal cost units. Verbose terminal output by default.
