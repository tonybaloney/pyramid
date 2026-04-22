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

Per-model **cost table** (relative units, scaled per 1k tokens; treat as
ordinal — only ratios matter):

| Tier | Claude track       | Cost | GPT track       | Cost |
|------|--------------------|------|------------------|------|
| 1    | claude-haiku-4.5   | 1    | gpt-5-mini       | 1    |
| 2    | claude-sonnet-4.6  | 5    | gpt-5.4          | 4    |
| 3    | claude-opus-4.7    | 25   | gpt-5.3-codex    | 20   |
| V    | claude-haiku-4.5   | 1    | gpt-5-mini       | 1    |

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
3. `expected_gain(N→N+1, confidence) * value_of_correctness >= cost(N+1) / cost(N)`.
   Use `value_of_correctness = 10` as the default (correctness is ~10× more
   valuable than the relative compute cost of tier-1).
4. `spent_so_far + estimated_cost(N+1) <= budget` (skip if no budget set).

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
  "estimated_cost_used": <number>
}
```
````

The orchestrator uses `decision` to either move on to the next DAG node
(accept) or re-dispatch the same subtask to `next_tier` (escalate).
