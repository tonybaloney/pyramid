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

## Baseline #1 — methodology bring-up (token-weighted UX release, 0.2.0)

**Status:** _placeholder — full 12 × 3 benchmark not yet executed_

**Why a placeholder:** the v0.2.0 PR ships the instrumentation, cost
model, and quiet-output UX needed to *measure* the benchmark, but a full
12-task × 3-run execution against live LLMs was deferred to a follow-up.
This section establishes the schema; numbers populate on the next run.

**Config (planned):**

```
--track=claude --theta=0.75,0.85 --anchor=off
--budget=50000 --output=quiet --value-of-correctness=8
--max-tier=3
```

**Per-task table (to be filled):**

| # | Task slug                          | Final tier | Verifier conf | Decision path | Tokens (in/out) | Cost (tw) | Pass? |
|---|------------------------------------|------------|---------------|---------------|------------------|-----------|-------|
| 1 | list-md-modified-7d                | —          | —             | —             | —                | —         | —     |
| 2 | rename-pyramid-verify-folder       | —          | —             | —             | —                | —         | —     |
| 3 | fix-broken-readme-links            | —          | —             | —             | —                | —         | —     |
| 4 | add-retry-budget-flag              | —          | —             | —             | —                | —         | —     |
| 5 | refactor-orchestrate-split-logging | —          | —             | —             | —                | —         | —     |
| 6 | investigate-anchoring-degradation  | —          | —             | —             | —                | —         | —     |
| 7 | design-plugin-sandbox              | —          | —             | —             | —                | —         | —     |
| 8 | generate-skill-api-docs            | —          | —             | —             | —                | —         | —     |
| 9 | verify-tools-frontmatter           | —          | —             | —             | —                | —         | —     |
| 10| quantum-verifier-adversarial       | —          | —             | —             | —                | —         | —     |
| 11| make-it-better-vague               | —          | —             | —             | —                | —         | —     |
| 12| dry-run-task5                      | n/a        | n/a           | dry-run       | 0 / 0            | 0         | —     |

**Aggregate metrics (to be filled):**

| Metric                                     | Target  | Measured |
|--------------------------------------------|---------|----------|
| Mean savings vs always-tier-3 (#1–9)       | ≥ 35%   | —        |
| Tier-1 accept-rate (#1–9)                  | ≥ 60%   | —        |
| Tier-3 dispatch rate (#1–6, #8, #9)        | ≤ 15%   | —        |
| Median terminal output lines (`quiet`)     | ≤ 5     | —        |
| Median report file size                    | n/a     | —        |
| Adversarial #10 hit max_tier and warned    | yes     | —        |
| Adversarial #11 terminated within budget   | yes     | —        |
| Dry-run #12 wrote trace, no dispatch       | yes     | —        |

**Open follow-ups discovered during bring-up:**

- Confirm GPT-track multipliers with the user before benchmarking the
  `gpt` and `mixed` tracks (currently placeholder values in
  `pyramid-verify`).
- Investigate whether the `task` tool surfaces token counts directly. If
  not, the `cost_estimated: true` path will be exercised and the savings
  numbers should be marked accordingly.
- Re-tune `value_of_correctness` empirically on the first real baseline.
  The 0.2.0 default of `8` was derived from the 1:3:21 ratio change but
  not yet validated against measured accept-rate / savings curves.
