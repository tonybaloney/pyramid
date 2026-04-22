---
name: pyramid-verify
description: Verify a tier-N solver's output and decide via Hansen-Zilberstein Value-of-Computation whether to escalate to tier N+1. Used by pyramid-orchestrate.
---

# pyramid-verify

Wraps a call to the `pyramid-verifier` agent and applies the Value-of-Computation
escalation rule. Returns a structured decision the orchestrator uses to
either accept the answer or escalate.

## Inputs

- `subtask`: the DAG node being evaluated.
- `tier`: the current tier (1, 2, or 3) that produced the answer.
- `answer`: the solver's `pyramid-result` block.
- `pyramid_config`: from `/pyramid` — includes `theta`, `max_tier`, `budget`,
  `spent_so_far`, and the resolved tier→model map.

## Step 1 — Call the verifier

Use the `task` tool with `agent_type: "general-purpose"`,
`model: "claude-haiku-4.5"`, `name: "pyramid-verify"`, `mode: "sync"`,
and a prompt that:

1. Tells the verifier it is the `pyramid-verifier` agent (paste the
   `pyramid-verifier.agent.md` body if the agent isn't auto-loaded).
2. Provides `subtask`, `answer`, and `tier`.
3. Repeats the strict-JSON output contract.

Parse the returned `pyramid-verdict` JSON. Fields: `confidence`, `issues`,
`would_benefit_from_escalation`, `evidence`.

## Step 2 — Apply the escalation rule (Hansen-Zilberstein VoC)

### Token-weighted cost model (v0.2.0)

Pyramid MoA accounts cost in **token-weighted units** (`tw-units`),
not flat ordinals. For any model call:

```
cost(call) = (input_tokens + output_tokens) * multiplier(model)
```

Per-model multipliers (per token, normalized so `claude-sonnet-4.6 = 1.0`):

| Tier | Claude track          | Multiplier | GPT track          | Multiplier  |
|------|-----------------------|------------|--------------------|-------------|
| 1    | `claude-haiku-4.5`    | 0.33       | `gpt-5-mini`       | 0.10 (TBD)  |
| 2    | `claude-sonnet-4.6`   | 1.00       | `gpt-5.4`          | 1.00 (TBD)  |
| 3    | `claude-opus-4.7`     | 7.00       | `gpt-5.3-codex`    | 5.00 (TBD)  |
| V    | `claude-haiku-4.5`    | 0.33       | `gpt-5-mini`       | 0.10 (TBD)  |

GPT-track multipliers are placeholders pending confirmation.

When the underlying `task` tool surfaces actual token counts, use them.
Otherwise fall back to a tokenizer estimate (`len(prompt + answer) / 4`)
and mark `cost_estimated: true` in the trace.

For the VoC pre-escalation comparison (where the next tier's call has
**not happened yet**), use a synthetic prior of **1500 input + 800 output
tokens** for the planned next-tier call, multiplied by that tier's
multiplier.

### VoC escalation rule

**Expected accuracy gain** from escalating tier N → N+1, given current
verifier confidence `c`:

```
expected_gain(N→N+1, c) = (1 - c) * P_correct(N+1 | N wrong)
```

Use these calibrated `P_correct(N+1 | N wrong)` priors from the paper
(Tables 1–3 generalized): tier1→2 = **0.55**, tier2→3 = **0.45**.

**Decision rule** — escalate iff ALL of:

1. `confidence < theta[N]` (theta defaults: θ₁=0.75, θ₂=0.85), **OR**
   `would_benefit_from_escalation == true` AND `confidence < theta[N] + 0.05`.
2. `tier < max_tier`.
3. `expected_gain(N→N+1, confidence) * value_of_correctness >= cost_ratio(N+1, N)`,
   where `cost_ratio(N+1, N) = projected_cost(N+1) / observed_cost(N)`
   using token-weighted costs from above. Default `value_of_correctness = 8`
   (re-tuned from `10` for the new 1:3:21 cost ratio; see
   `docs/benchmarks/baseline-metrics.md`).
4. `spent_so_far + projected_cost(N+1) <= budget` in tw-units (skip if no
   budget set). Note: `--budget` values are now interpreted as tw-units, not
   USD — the flag was previously documented as USD but always operated on
   ordinal units; this clarifies it.

Otherwise **accept** the current answer as best-so-far.

## Output contract

Return exactly this fenced block to the orchestrator:

````
```pyramid-decision
{
  "subtask_id": "<id>",
  "verifier_confidence": 0.0-1.0,
  "verifier_issues": ["..."],
  "decision": "accept" | "escalate",
  "next_tier": null | 2 | 3,
  "reason": "<one sentence: why this decision per VoC>",
  "estimated_cost_used": <number>,
  "cost_breakdown": {
    "solver_input_tokens": <int>,
    "solver_output_tokens": <int>,
    "solver_multiplier": <float>,
    "verifier_input_tokens": <int>,
    "verifier_output_tokens": <int>,
    "verifier_multiplier": <float>,
    "cost_estimated": <bool>
  }
}
```
````

`estimated_cost_used` is in **token-weighted units** (tw-units) and equals
`solver_call_cost + verifier_call_cost` for this verification step.
`cost_breakdown` lets the orchestrator and `pyramid-aggregate` reconstruct
totals and compute the always-tier-3 baseline by re-weighting tokens.

The orchestrator uses `decision` to either move on to the next DAG node
(accept) or re-dispatch the same subtask to `next_tier` (escalate).
