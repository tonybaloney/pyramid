# Pyramid MoA — baseline metrics

This file is updated after every benchmark run. Each section captures one
benchmark execution: config, methodology delta vs `tasks.md`, raw metrics,
and a before/after comparison against the prior baseline.

## How to read this file

- Source of truth is the per-run JSON trace under
  `~/.copilot/session-state/<id>/files/pyramid-runs/<task>-<ts>.json`.
- Add a new section at the top for each new run; never overwrite
  historical sections.
- "Before" column = the previous published baseline; "After" = the run you
  are reporting.

---

## Baseline #1 — smoke run (3 tasks × 1 run, v0.2.0, 2026-04-23)

**Status:** _smoke benchmark only — 3 of 12 tasks, 1 run each (no median).
Surfaced 4 real defects that should be fixed before the full 12 × 3 run._

**Methodology delta vs `tasks.md`:** subset = `{#1, #3, #4}`; n=1 per task;
no `--max-tier=1` control collected; `always-T3` cost computed analytically
from captured token totals × Opus multiplier (7.00) per `tasks.md` step 3.
Each task invoked via `copilot -p "/pyramid:pyramid <task>" --allow-all-tools
--autopilot` from a fresh shell, with the v0.2.0 plugin installed from
`origin/main` at commit `17d790f`.

**Per-task table:**

| # | Task slug                  | Subtasks | Final tier | Verifier conf | Decision path                    | Tokens (in/out) | Cost (tw)     | Always-T3 (tw) | Saved   | Wall-clock | Pass?    |
|---|----------------------------|----------|------------|---------------|----------------------------------|-----------------|---------------|----------------|---------|------------|----------|
| 1 | list-md-modified-7d        | 1        | T1         | 0.92          | T1 → accept (verifier skipped)   | 600 / 300       | 297.0         | 6,300          | 95.3 %  | 134 s      | **fail**¹ |
| 3 | fix-broken-readme-links    | 3        | T1         | 0.95          | T1 → accept (2 subtasks inlined) | 1,100 / 350     | **0.5**²      | 10,150         | 99.99 %²| 163 s      | **fail**³ |
| 4 | add-retry-budget-flag      | 2        | T1         | 0.82          | T1 → accept (no escalation)      | ~5,000 / ~6,000⁴| _no JSON_⁵    | _no JSON_⁵     | _n/a_   | 499 s      | **fail**⁴ |

¹ Acceptance gate "verifier confidence ≥ 0.80" met, but the verifier was
   never actually called — the orchestrator emitted the verdict directly.
   See defect **B3** below.
² **Cost arithmetic bug** — see **B1** below. Correct value ≈ 478.5 tw-units.
³ Per-task gate "tier-1 accept; no escalations" *technically* met, but only
   1 of 3 subtasks was actually dispatched. See defect **B4** below.
⁴ JSON trace missing — see **B2** below. Token counts estimated from solver
   output size in the Markdown report.
⁵ Aggregate skill wrote only the `.md` report; no companion `.json` trace.

**Aggregate metrics (smoke subset, n=3):**

| Metric                                     | Target  | Measured     |
|--------------------------------------------|---------|--------------|
| Mean savings vs always-tier-3              | ≥ 35 %  | 96.8 %²      |
| Tier-1 accept-rate                         | ≥ 60 %  | 100 %        |
| Tier-3 dispatch rate                       | ≤ 15 %  | 0 %          |
| Median terminal output lines (`quiet`)     | ≤ 5     | ~12⁶         |
| Median wall-clock per task                 | n/a     | 163 s        |
| Median premium-request count per task      | n/a     | 7.5          |
| JSON trace written                         | 100 %   | 67 % (2/3)   |
| Verifier dispatched on every subtask       | 100 %   | 33 % (1/3)   |

⁶ Quiet *summary* line is 2 lines as designed, but the agent also dumps the
   final answer body (markdown headers, bullets, tables) below it. See **C3**.

### Defects discovered (must fix before full 12 × 3 run)

- **B1 — Cost arithmetic inconsistent.** Task 3 reported `cost_tw_units = 0.5`
  in the JSON trace and the savings line. The correct value, per the
  formula in `pyramid-verify` § Cost model, is
  `(600+250) × 0.33 + (500+100) × 0.33 ≈ 478.5`. The orchestrator appears
  to have stored an `estimated_cost_used` value (which has been a
  per-verdict rough scalar, not a tw-unit quantity) directly into the
  trace `totals.cost_tw_units` field. Fix: have `pyramid-orchestrate`
  always recompute `totals.cost_tw_units` from `subtasks[*].calls[*]
  .{input_tokens, output_tokens, model}`; ignore any per-verdict
  `estimated_cost_used` heuristic when totaling.
- **B2 — JSON trace not written.** Task 4 produced only
  `design-how-to-add-a-retry-budget-n-flag-…md`; no `.json`. The
  pairing is mandatory per `pyramid-aggregate` SKILL. Fix: tighten the
  output contract — the JSON write should be the *first* step in
  `pyramid-aggregate`, before the Markdown render, and aggregate must
  refuse to print the quiet summary if the JSON write failed.
- **B3 — Verifier short-circuit.** On task 1 the orchestrator emitted a
  `pyramid-decision` block itself ("verifier round-trip would cost more
  than the work itself") and never dispatched `pyramid:pyramid-verifier`.
  No skill authorizes this. Fix options: (a) make verifier dispatch
  mandatory in `pyramid-orchestrate` step "for each subtask"; (b) or, if
  we want a "trivial-task fast-path", add it to `pyramid-verify` as an
  explicit, named heuristic with conditions and a trace flag
  (`verifier_skipped: true`, `verifier_skip_reason: "..."`).
- **B4 — Inline-resolved subtasks.** Task 3 had a 3-node DAG; only the
  first node was dispatched to a tier-1 solver. The other two were
  "resolved inline" by the orchestrator. Same root cause as B3 — the
  orchestrator is freelancing. Fix: dispatch must be unconditional for
  every DAG node (or, again, add an explicit "resolve-inline" path to
  `pyramid-verify` with a trace flag and a confidence rule).

### Calibration findings (informational, no fix required this PR)

- **C1 — Tier-1 (Haiku-4.5) is over-priored.** Task 4's design subtask had
  a `difficulty_prior` of 0.65 (chosen specifically to *expect* escalation
  per `tasks.md` task #4) and was accepted at tier-1 with confidence 0.82.
  Either Haiku-4.5 is more capable than the paper's tier-1 baseline, or
  the verifier is being lenient. Re-tune `θ₁` and / or
  `pick_initial_tier` thresholds after the full run.
- **C2 — Wall-clock overhead is high for cheap tasks.** Task 1 (a 14-line
  `git log` invocation) took 134 s and consumed 7.5 premium requests
  through the orchestrator chain (decompose + solver + missed-verifier +
  aggregate). The router only saves money if the saved tier-3 spend
  exceeds this overhead. Worth measuring orchestrator cost separately
  in a future revision (`overhead_tw_units` totals field).
- **C3 — Quiet is not actually quiet.** The 2-line summary line is rendered
  correctly, but the host agent also dumps the synthesized final answer
  (markdown sections, bullets, tables) into the terminal beneath it.
  Quiet mode should suppress everything except the 2-line summary
  + ⚠ warnings + the report path. Fix: tighten the contract in
  `pyramid-aggregate` step "Output verbosity" — at quiet, do not return
  the answer body to the host agent at all; only the 2-line block.
- **C4 — Token counts always estimated.** Confirmed: every call recorded
  `cost_estimated: true`. The host `task` tool does not surface
  per-call token counts to the orchestrator. The fallback
  `len(prompt+answer)/4` heuristic appears to be off by ~3-10× in
  practice (compare 600 in / 300 out for a 14-line answer that the
  outer Copilot CLI billed at 518 k input / 5.8 k output). The
  estimator should either (a) sample and calibrate, or (b) be
  documented as directional-only.

### Open follow-ups (carried over)

- Confirm GPT-track multipliers with the user before benchmarking the
  `gpt` and `mixed` tracks.
- Re-tune `value_of_correctness` empirically on the next real baseline.
- Run the remaining 9 tasks (×3 each) once B1–B4 are fixed.
