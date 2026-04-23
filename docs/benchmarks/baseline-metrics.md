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

## Baseline #2 — full benchmark (12 tasks × 1 run, v0.2.1, 2026-04-23)

**Status:** _full 12-task suite, 1 run per task (no median per `tasks.md` §
methodology, deferred for cost). Adversarial tasks (#10, #11) excluded from
savings averages per the spec but included for terminate-cleanly grading._

**Methodology delta vs `tasks.md`:** n=1 per task (not 3); no `--max-tier=1`
control collected (always-T3 baseline computed analytically from captured
token totals × Opus multiplier 7.00). Each task invoked via
`copilot -p "/pyramid:pyramid <task>" --allow-all-tools --autopilot` from a
fresh shell, in 3 staggered batches of 4 tasks parallel each. Plugin
installed from `origin/main` at commit `37ca093` (v0.2.1).

**Per-task table:**

| # | Task slug                              | Subtasks | T1 / T2 / T3 disp. | Vrf. | Tokens (in/out)  | Cost (tw)     | Always-T3 (tw) | Saved   | Wall-clock | Gate    |
|---|----------------------------------------|----------|--------------------|------|------------------|---------------|----------------|---------|------------|---------|
| 1 | list-md-modified-7d                    | 1        | 1 / 0 / 0          | 1    | 300 / 360        | 217.8         | 4,620          | 95.3 %  | 30 m 08 s¹ | PASS    |
| 2 | rename-pyramid-verify-skill            | _no JSON_| _no JSON_          | _na_ | _no JSON_²       | _no JSON_²    | _no JSON_²     | _n/a_   | 32 m 15 s¹ | partial² |
| 3 | fix-broken-readme-links                | 1        | 1 / 0 / 0          | 1    | 1,300 / 570      | 617.1         | 13,090         | 95.3 %  | 31 m 47 s¹ | PASS    |
| 4 | add-retry-budget-flag                  | 1        | 1 / 0 / 0          | 1    | 2,600 / 1,750    | 1,435.5       | 30,450         | 95.3 %  | 32 m 40 s¹ | PARTIAL³ |
| 5 | refactor-orchestrate-split-logging     | 1        | 0 / 1 / 0          | 1    | 1,450 / 1,980    | 3,430.0       | 24,010         | 85.7 %  | 5 m 05 s   | PASS    |
| 6 | anchoring-degrades-accuracy            | 1        | 1 / 0 / 0          | 1    | 1,120 / 480      | 528.0         | 11,200         | 95.3 %  | 2 m 33 s   | PASS    |
| 7 | sandbox-untrusted-plugins              | 7        | 4 / 3 / 0          | 8    | 33,600 / 33,400  | 36,912.0      | 469,000        | 92.1 %  | 35 m 58 s¹ | PARTIAL⁴ |
| 8 | api-docs-for-skills                    | 2        | 2 / 0 / 0          | 2    | 0 / 0            | **0.0**⁵      | **0.0**⁵       | 0 %⁵    | 3 m 11 s   | PARTIAL⁵ |
| 9 | verify-agent-tools-frontmatter         | _no JSON_| _bypassed_⁶        | _na_ | 118 k host-side⁶ | _bypassed_⁶   | _bypassed_⁶    | _n/a_   | 1 m 14 s   | FAIL⁶   |
| 10| quantum-resistant-verifier (adversarial)| _killed_⁷| _killed_⁷         | _na_ | _killed_⁷        | _killed_⁷     | _killed_⁷      | _n/a_   | _killed_⁷  | FAIL⁷   |
| 11| "make it better" (adversarial)         | _killed_⁷| _killed_⁷         | _na_ | _killed_⁷        | _killed_⁷     | _killed_⁷      | _n/a_   | _killed_⁷  | FAIL⁷   |
| 12| dry-run of #5                          | _no JSON_| _bypassed_⁶        | _na_ | 103 k host-side⁶ | _bypassed_⁶   | _bypassed_⁶    | _n/a_   | 1 m 30 s   | FAIL⁶   |

¹ Wall-clock inflated by 4-way parallel batches contending for Copilot API
   throughput; the smoke baseline at v0.2.0 measured the same workloads at
   ~2–8 min when run sequentially. Cost figures (premium requests, tokens)
   are unaffected by parallelism — see "Findings" below.
² Task 2 regression: orchestrator dispatched a single subtask, then the host
   agent dumped the full ~1.2 MB grep-and-summary answer body inline at
   `--output=quiet`, never reached `pyramid-aggregate`, no JSON trace
   written. C3 contract violated under high token-count workloads.
³ Per-task gate required at least one escalation; task accepted at T1
   instead. Same pattern as #1: tier-1 (Haiku) handled it cleanly so the
   verifier accepted — calibration finding C1 from Baseline #1 reconfirmed.
⁴ Per-task gate required final tier = 3; sandboxing design accepted at T2
   for every subtask. The decompose split it into 7 small subtasks each
   solvable in T1/T2, so the gate is mis-calibrated rather than the run.
⁵ **New defect B5:** when subtask work happens entirely in the host
   context (no `task` tool dispatch), token counts are estimated as 0,
   collapsing cost to 0. Surfaced cleanly via `cost_estimated: true` and a
   `⚠` warning, but the savings number is meaningless.
⁶ **New defect B6:** for "trivial" tasks the host agent occasionally
   bypasses `pyramid-orchestrate` entirely and answers in 1 premium
   request. No DAG, no verifier, no trace. The pipeline is silently
   skipped — the user has no way to know they didn't get MoA routing.
⁷ **Adversarial gate failure:** tasks 10 and 11 are designed to test
   `max_tier`-without-accept and decomposition robustness on vague prompts.
   The orchestrator instead began making real code edits to `agents/`,
   `skills/`, `AGENTS.md`, `commands/pyramid.md`, etc. Killed manually and
   reverted via `git stash`. Per-task gate "subtask reaches max_tier" failed.

**Aggregate metrics (non-adversarial subset, n=7 with traces):**

| Metric                                     | Target  | Measured     |
|--------------------------------------------|---------|--------------|
| Mean savings vs always-tier-3 (#1,3,4,5,6,7) | ≥ 35 % | **93.2 %**¹ |
| Tier-1 accept rate (#1,3,4,6,7-T1 nodes)   | ≥ 60 %  | 83 % (5/6)   |
| Tier-3 dispatch rate (excl. #7,#10)        | ≤ 15 %  | **0 %**      |
| Median terminal output lines at quiet      | ≤ 5     | **2 (#1,3,4,7,8)** ; ⚠ leaked on #2,#5,#6 |
| `pyramid-verify` invoked on every dispatch | yes     | **yes** (B3 fix holding for traced runs) |
| JSON trace written on every traced run     | yes     | **5 of 7** (#2,#9,#12 missing — bypass/regression) |

¹ Excludes #8 (B5 zero-token bug) and the 5 tasks without traces.

**Findings:**

- **B1 cost arithmetic — FIXED.** All 7 traced runs report cost numbers
  that match recomputation from per-call tokens. Baseline #1 had reported
  a per-verdict scalar (`0.5`) for task #3; v0.2.1 reports `617.1` for the
  same workload.
- **B2 missing JSON trace — FIXED for the orchestrated path.** When
  `pyramid-orchestrate` runs end-to-end the JSON trace appears (5/5).
  Tasks #2, #9, #12 still lack JSON because the orchestrator was bypassed
  upstream (B6) or the answer body was dumped inline (C3 leak on #2).
- **B3 verifier short-circuit — FIXED.** Every traced run shows
  `verifier_calls ≥ 1`. Tasks #1 / #3 / #4 each show exactly 1 verifier
  call per subtask (no orchestrator-emitted decision blocks).
- **B4 inline-resolved subtasks — FIXED.** Every node in `dag.nodes` is
  represented in `tier_dispatches`; no node skipped. Task #7 dispatched
  all 7 of its DAG nodes.
- **C3 quiet output — PARTIALLY FIXED.** The 2-line summary template now
  appears verbatim on tasks #1, #3, #4, #7, #8, but the host agent still
  appends an answer body on tasks #2, #5, #6. The contract is in the
  SKILL but the model treats it as advisory. Likely needs a stronger
  framing (e.g., command-level post-condition) — opened as candidate for
  v0.2.2.
- **B5 (new) — host-context tokens collapse to 0.** Task #8 split into
  2 subtasks the host completed itself; trace recorded `tokens=0` and
  `cost=0`. The `⚠` warning surfaces but the cost number is misleading.
- **B6 (new) — orchestrator-bypass on trivial tasks.** Tasks #9 and #12
  used 1 premium request total and emitted no `pyramid-*` blocks. The
  host agent decided "this is too small to MoA" and answered directly.
  The pipeline must either (a) refuse to bypass at the command level,
  or (b) emit a `⚠ pipeline bypassed` warning so the user knows.
- **Adversarial gate failure (#10, #11).** Both tasks' purpose is to test
  refusal / max-tier behavior. Instead the orchestrator (in `--autopilot`)
  began real code edits. Need a `--read-only` invariant or a hard
  "synthesize tasks must not edit files outside the report path" rule.
- **Wall-clock inflation under parallelism.** Tasks run in 4-way parallel
  batches took 30+ minutes each vs 2–8 min in the v0.2.0 sequential
  smoke. Per-call premium request count (7.5) is unchanged, so the
  bottleneck is API queueing, not pipeline depth.

**v0.2.1 vs v0.2.0 — defect-fix matrix:**

| Defect | Baseline #1 (v0.2.0) | Baseline #2 (v0.2.1) | Status |
|--------|----------------------|----------------------|--------|
| B1     | task #3: 0.5 (wrong) | task #3: 617.1 (correct) | ✓ fixed |
| B2     | task #4: no JSON      | 5/5 traced runs have JSON | ✓ fixed (when orchestrated) |
| B3     | task #1: 0 verifier   | task #1: 1 verifier, all subtasks verified | ✓ fixed |
| B4     | task #3: 1 of 3 dispatched | task #7: 7 of 7 dispatched | ✓ fixed |
| C3     | answer body always dumped | dumped on 3 of 7 (#2,#5,#6) | partial |
| B5     | _not surfaced_        | task #8: cost=0 on host-context work | new |
| B6     | _not surfaced_        | tasks #9, #12: pipeline bypassed | new |

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
