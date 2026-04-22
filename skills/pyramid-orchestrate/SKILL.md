---
name: pyramid-orchestrate
description: Main control loop for Pyramid MoA. Decomposes a task, dispatches each subtask to the cheapest tier, verifies, and escalates per Hansen-Zilberstein Value-of-Computation rule.
---

# pyramid-orchestrate

This is the heart of the Pyramid MoA plugin. The `/pyramid` slash command
hands control here with a `pyramid_config` and the user's task description.

## Inputs

- `pyramid_config` — resolved by `/pyramid`:
  - `track_models`: `{1: "<tier1 model id>", 2: "<tier2>", 3: "<tier3>", "V": "<verifier>"}`
  - `theta`: `[θ₁, θ₂]` (default `[0.75, 0.85]`)
  - `max_tier`: `1 | 2 | 3` (default `3`)
  - `anchor`: `true | false` (default `false`)
  - `budget`: USD cap or `null`
  - `dry_run`: `true | false` (default `false`)
- `task`: free-text user task description.

## Algorithm

```
ledger = []                          # per-subtask trace
spent_so_far = 0.0
results = {}                          # subtask_id -> final pyramid-result

dag = invoke skill pyramid-decompose with task         # returns pyramid-dag block
order = topological_sort(dag.nodes)

if pyramid_config.dry_run:
    print_dag_and_planned_tiers(dag, pyramid_config)
    return  # no dispatch, no aggregate

for node in order:
    tier = pick_initial_tier(node.difficulty_prior)    # see below
    last_answer = null
    last_verdict = null
    while True:
        ctx = build_context(node, results, last_verdict, anchor, last_answer)
        answer = dispatch(tier, node, ctx)              # task tool, model override
        verdict = invoke pyramid-verify(node, tier, answer, pyramid_config + spent_so_far)
        spent_so_far += verdict.estimated_cost_used

        ledger.append({node, tier, verdict.verifier_confidence, verdict.decision, verdict.reason})

        if verdict.decision == "accept" or tier == pyramid_config.max_tier:
            results[node.id] = answer
            break
        tier = verdict.next_tier
        last_answer = answer
        last_verdict = verdict

invoke pyramid-aggregate with (results, ledger, dag, task)
```

### `pick_initial_tier(prior)`

| `difficulty_prior` | Initial tier |
|--------------------|--------------|
| `< 0.40`           | 1            |
| `0.40 – 0.70`      | 1            |
| `> 0.70`           | 2 (skip tier-1 — paper §3 shows tier-1 is wasted compute on hard subtasks) |

`max_tier` always wins as a hard ceiling.

### `dispatch(tier, node, ctx)`

Call the `task` tool with:

- `agent_type`: `"general-purpose"`
- `model`: `pyramid_config.track_models[tier]`
- `name`: `f"pyramid-tier{tier}-{node.id}"`
- `mode`: `"sync"`
- `description`: `f"Pyramid MoA tier-{tier} solving subtask {node.id}"`
- `prompt`: see template below.

Prompt template:

```
You are the tier-{tier} solver in a Pyramid MoA pipeline. Apply the rules
in agents/tier{tier}-*.agent.md (load them if not already in context).

Subtask:
  id: {node.id}
  kind: {node.kind}
  description: {node.description}
  success_criteria: {node.success_criteria}

Dependencies already completed:
{for dep_id in node.dependencies: results[dep_id].answer summarized}

{if last_verdict and not anchor:}
A previous tier attempted this subtask and the verifier flagged these issues:
  {last_verdict.verifier_issues}
You are NOT shown the previous answer (anchoring mitigation). Solve fresh.

{if last_verdict and anchor:}
Previous tier-{tier-1} attempt (verifier confidence {last_verdict.verifier_confidence}):
  {last_answer}
Verifier issues: {last_verdict.verifier_issues}
Improve on it.

Return your result in the pyramid-result fenced JSON block exactly as specified.
```

Parse the returned `pyramid-result` block from the agent's output.

### `build_context(node, results, last_verdict, anchor, last_answer)`

Collects:
- The completed answers of all `node.dependencies`, summarized (do not paste
  full code — just the agreed-upon outputs / file paths edited).
- The verifier's prior critique if escalating.
- The prior tier's answer **only if** `anchor == true` (default: omit).

## Anchoring discipline

Paper §4 shows that passing **incorrect** lower-tier reasoning to a higher
tier degrades accuracy by up to **−18.0pp** across MBPP/GSM8K/MMLU/HumanEval.
Passing **correct** lower-tier reasoning improves it by **+19.2pp**, but we
cannot reliably distinguish correct from incorrect lower-tier output (that's
literally why we escalated). So default behavior is `anchor=off`. Document
this in the ledger as `anchor_policy`.

## Dry-run mode

When `pyramid_config.dry_run` is `true`, the orchestrate skill validates the decomposition and tier strategy without invoking any tier solvers, verifiers, or the aggregate skill. 

**Trigger:** The `--dry-run=on` flag is set in the `/pyramid` slash command.

**Behavior:**
1. After `pyramid-decompose` returns the DAG and `topological_sort` is called, an early return occurs.
2. No dispatcher, verifier, or aggregate skills are invoked.
3. The following is printed:
   - The decomposed DAG structure (all nodes and their dependencies)
   - For each node in topological order, the planned initial tier as determined by `pick_initial_tier(node.difficulty_prior)` along with its `difficulty_prior` value

**Example output format:**

```
=== Pyramid Dry-run ===
DAG has 4 nodes.

Node details:
| Subtask ID | Kind | difficulty_prior | Planned initial tier | Dependencies |
|------------|------|------------------|----------------------|--------------|
| sub-001 | synthesize | 0.35 | 1 | — |
| sub-002 | read | 0.50 | 1 | sub-001 |
| sub-003 | synthesize | 0.82 | 2 | sub-001, sub-002 |
| sub-004 | verify | 0.45 | 1 | sub-003 |

Dry-run complete. No tiers, verifiers, or aggregation were invoked.
```

## Output

After the loop completes, invoke the `pyramid-aggregate` skill with:
- `results`: map of subtask id → final answer
- `ledger`: full per-subtask trace
- `dag`: the original DAG
- `task`: original user task
- `spent_so_far`: total estimated cost

The aggregate skill produces the user-facing response.

## Failure handling

- If `pyramid-decompose` returns malformed JSON, retry once with a stricter
  prompt; if still bad, fall back to a single-node DAG containing the raw task.
- If a tier solver returns no `pyramid-result` block, treat as
  `self_confidence=0.0, blocked_reason="malformed output"` and feed to verify
  (which will return low confidence → escalate or accept at max_tier).
- If verifier returns malformed JSON, treat as `confidence=0.5,
  would_benefit_from_escalation=true` and proceed conservatively.
- Infinite-loop guard: at most **2 escalations per subtask** (tier 1→2→3),
  enforced by the `tier == max_tier` exit condition.
