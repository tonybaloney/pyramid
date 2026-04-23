# Changelog

All notable changes to this plugin are documented here.

## 0.2.3 — Fix C3-residual quiet-mode leakage (#7)

### Fixed

- **C3 residual (#7)** — While v0.2.1 introduced the hard quiet contract,
  ~3 of 7 baseline tasks still leaked answer-body text, headings, lists, or
  tables to the terminal at `--output=quiet`. Root cause: the quiet
  contract was described in `pyramid-aggregate` but not enforced as a
  command-level post-condition, and the host agent assumed the contract was
  advisory rather than mandatory.

  Three changes:

  1. `commands/pyramid.md` now includes a new "Post-conditions (hard
     contract)" section at the end, restating the quiet rule in imperative
     voice ("MUST", "MUST NOT") and cross-referencing the aggregate skill.
  2. `skills/pyramid-aggregate/SKILL.md` — Updated both the quiet and
     edge-case output templates to add a third line: `(Answer body
     suppressed — pass --output=normal to see it here.)`. Added a new
     "Quiet contract violation detection" subsection instructing the host
     to prepend a `⚠ Quiet contract violated by host…` warning if it has
     already leaked any body content in the same turn (self-attestation).
  3. `plugin.json` version bumped to 0.2.3.

### Notes

- The quiet contract is now treated as a hard safety property, not a hint.
- This release is backward-compatible with v0.2.x trace consumers; no
  schema changes beyond cosmetic output.
- The "violated by host" warning is retroactive (it does not prevent the
  leak, only discloses it) to avoid host-side buffering complexity.

## 0.2.2 — fix B5 (cost-collapse on host-context calls)

### Fixed

- **B5 (#5)** — When a subtask's LLM call did not surface token counts
  (the `task` tool's typical case), the orchestrator's fallback estimator
  computed `len(prompt or "") / 4`. If the host completed work in its own
  context the prompt and answer text were both empty strings, collapsing
  the call's tokens — and therefore `cost_tw_units` and
  `cost_tw_units_always_tier3` — to `0.0`. The quiet summary then printed
  `cost 0.0 tw-units (vs 0.0 always-T3 → 0% saved)`, which is
  mathematically defensible but actively misleading. (Surfaced in
  Baseline #2, task #8.)

  Two changes:

  1. `skills/pyramid-orchestrate/SKILL.md` § "Capturing call metadata"
     pins a per-call floor: `MIN_TOKENS_PER_CALL = 50`, applied to both
     `input_tokens` and `output_tokens` in the unmeasured-tokens fallback.
     A genuinely no-dispatch subtask must produce `subtask.calls = []`,
     not a synthetic zero-token call.
  2. `compute_totals` now returns `savings_pct = None` (instead of `0`)
     when `cost_tw_units_always_tier3 == 0`. The aggregate skill's quiet
     template substitutes a `cost not measurable (host context)` clause
     and prepends a `⚠` warning rather than printing the misleading
     `0% saved`.

### Notes

- No trace-shape changes beyond `savings_pct` being nullable. Existing
  trace consumers should treat `None`/`null` as "undefined", not zero.
- B6 (issue #6, orchestrator-bypass) and the residual C3 leak (issue #7)
  are out of scope for this release.

## 0.2.1 — smoke-benchmark fixes (B1–B4, C3)

### Fixed

- **B1 — Cost arithmetic.** `pyramid-orchestrate` now pins
  `compute_totals` as the single source of truth for cost numbers, with
  an explicit Python-style formula and a sanity assertion that
  `totals.cost_tw_units` matches the per-call recomputation. Per-verdict
  `estimated_cost_used` scalars are documented as informational only —
  never substituted into totals. Smoke run had reported
  `cost_tw_units = 0.5` for a task whose true value was ≈ 478.5.
- **B2 — JSON trace must be written.** `pyramid-aggregate` now verifies
  the JSON trace exists as Step 1 (pre-condition) and writes it itself
  with a `⚠` warning if `pyramid-orchestrate` failed to. The orchestrate
  skill now also explicitly contracts that `write_trace` runs before
  invoking aggregate.
- **B3 — Verifier short-circuit.** `pyramid-orchestrate` now contracts
  that every dispatched answer must go through `pyramid-verify`. No
  "verifier round-trip would cost more than the work" shortcut. Any
  fast-path must live inside `pyramid-verify` itself with a named
  heuristic and a `verifier_skipped: true` trace flag.
- **B4 — Inline-resolved subtasks.** `pyramid-orchestrate` now contracts
  that every DAG node must be dispatched. No "resolve inline" shortcut.
- **C3 — Quiet is now actually quiet.** `pyramid-aggregate` Step 4 adds
  a hard contract: at `output_level=quiet`, the terminal output is
  exactly `⚠` warning lines plus the two-line summary. The host agent
  must NOT append the synthesized answer body, headers, or tables.

### Notes

- No behavior changes for `--output=normal` or `--output=debug`.
- No trace-shape changes; this release is backward-compatible with
  v0.2.0 trace consumers.
- Discovered via the v0.2.0 smoke benchmark (3 tasks). See
  `docs/benchmarks/baseline-metrics.md` § Baseline #1 for raw evidence.

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
