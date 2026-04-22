# Pyramid MoA — standard benchmark suite

This file defines the canonical task suite used to measure Pyramid MoA's
routing efficiency, escalation behavior, and developer-UX overhead. Re-run
after non-trivial changes to any agent, skill, or command and update
`baseline-metrics.md`.

## Methodology

For each task:

1. Run with default config: `--track=claude --theta=0.75,0.85 --anchor=off
   --budget=50000 --output=quiet --value-of-correctness=8`.
2. Run a control with `--max-tier=1` to establish "always-tier-1" cost.
3. Compute always-tier-3 reference analytically from the captured token
   totals × Opus multiplier (7.00). No actual tier-3 dispatch needed.
4. Repeat each run **3×** and report the median for each metric (LLM
   nondeterminism).
5. Tasks are read-only or hypothetical against this repo; **no real file
   edits are applied** during benchmarking.

All runs read the JSON trace under
`<session>/files/pyramid-runs/` to derive the metrics in
`baseline-metrics.md`. The trace is the single source of truth.

## Task table

| #  | Task                                                                  | Kind          | Expected difficulty | Stress target                                  |
|----|-----------------------------------------------------------------------|---------------|---------------------|------------------------------------------------|
| 1  | "List all Markdown files modified in last 7 days"                     | read          | low (0.10)          | tier-1 single-shot baseline                    |
| 2  | "Rename the `pyramid-verify` skill folder to `pyramid-verifier-skill`"| transform     | low (0.20)          | tier-1, multi-file edit (hypothetical)         |
| 3  | "Fix any markdown link that points at a non-existent path in README"  | transform     | medium (0.40)       | tier-1, expected accept                        |
| 4  | "Add a `--retry-budget=N` flag to `/pyramid:pyramid`"                 | synthesize    | medium (0.55)       | tier-1, expected escalation to tier-2          |
| 5  | "Refactor pyramid-orchestrate to split orchestration from logging"    | synthesize    | high (0.80)         | should skip tier-1 → tier-2/3                  |
| 6  | "Investigate why anchoring=on degrades accuracy per the paper"        | synthesize    | high (0.70)         | debug-style path; ambiguity bumps              |
| 7  | "Design a sandboxing model so untrusted plugins can't read this repo" | synthesize    | very high (0.90)    | exercises tier-3 path                          |
| 8  | "Generate API-style docs for the four pyramid-* skills"               | write         | medium (0.40)       | tier-1, large output                           |
| 9  | "Verify that every `tools` entry in agent frontmatter is a real tool" | verify        | medium (0.30)       | tier-1, exhaustive check                       |
| 10 | "Implement a quantum-resistant verifier" (adversarial)                | synthesize    | trick (0.95)        | should hit `max_tier`                          |
| 11 | "Make it better" (adversarial vague)                                  | synthesize    | trick (high+ambig)  | tests `pyramid-decompose` robustness           |
| 12 | Task #5 with `--dry-run=on`                                           | dry-run       | n/a                 | path coverage for dry-run + trace write        |

## Per-task acceptance gates

| #  | Pass criteria                                                                                  |
|----|------------------------------------------------------------------------------------------------|
| 1  | Final tier ≤ 2; verifier confidence ≥ 0.80; ≤ 1 escalation.                                    |
| 2  | Tier-1 accept; no escalations.                                                                 |
| 3  | Tier-1 accept; no escalations.                                                                 |
| 4  | At least one escalation triggered; final tier 2 or 3; final answer mentions a flag-parsing edit. |
| 5  | Initial tier ≥ 2 (skip tier-1 path); final tier ≥ 2.                                           |
| 6  | Final answer references "−18 pp" or paper §4; final tier ≤ 2.                                  |
| 7  | Final tier = 3; non-empty answer; budget not exhausted.                                        |
| 8  | Tier-1 accept; output ≥ 4 doc sections.                                                        |
| 9  | Tier-1 accept; verifier evidence references at least one frontmatter file.                     |
| 10 | Subtask reaches `max_tier`; orchestrator emits `⚠ max_tier without accept`.                    |
| 11 | DAG has ≥ 2 nodes; no infinite loop; total cost ≤ budget.                                      |
| 12 | Dry-run prints DAG; no solver/verifier dispatched; trace JSON written with `dry_run: true`.    |

## Aggregate acceptance gates

Across the non-adversarial subset (#1–#9):

- **Mean savings vs always-tier-3 ≥ 35%** (paper claims up to 42.9%).
- **Tier-1 accept-rate ≥ 60%**.
- **Tier-3 dispatch rate ≤ 15%** (excluding tasks #7 and #10 which target it).
- **Median terminal-output line count ≤ 5** at default `--output=quiet`
  (UX gate; full report file preserves all detail).

Adversarial tasks (#10, #11) are excluded from savings averages but must
still terminate cleanly within `--budget=50000`.
