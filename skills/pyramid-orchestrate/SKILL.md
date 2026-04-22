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
  - `budget`: token-weighted-unit cap or `null` (was previously documented as USD; was always ordinal)
  - `dry_run`: `true | false` (default `false`)
  - `output_level`: `quiet | normal | debug` (default `quiet`)
  - `value_of_correctness`: float (default `8`, see `pyramid-verify`)
  - `session_dir`: absolute path to the current session-state folder, used for trace + report files
- `task`: free-text user task description.

## Algorithm

```
ledger = []                          # per-subtask trace (rich)
spent_so_far = 0.0                   # token-weighted units (tw-units)
results = {}                          # subtask_id -> final pyramid-result
trace = {                             # written to disk at the end
    "task": task,
    "config": pyramid_config,
    "started_at": now_iso(),
    "subtasks": [],                  # one entry per DAG node
    "totals": {}                     # filled at end
}

dag = invoke skill pyramid-decompose with task         # returns pyramid-dag block
trace["dag"] = dag
order = topological_sort(dag.nodes)

if pyramid_config.dry_run:
    print_dag_and_planned_tiers(dag, pyramid_config)
    write_trace(trace, pyramid_config.session_dir, dry_run=True)
    return  # no dispatch, no aggregate

for node in order:
    tier = pick_initial_tier(node.difficulty_prior)
    last_answer = null
    last_verdict = null
    subtask_trace = {"id": node.id, "tier_path": [], "verdicts": [], "calls": []}
    while True:
        ctx = build_context(node, results, last_verdict, anchor, last_answer)
        t0 = now_ms()
        answer, call_meta = dispatch(tier, node, ctx)   # call_meta: input_tokens, output_tokens, model
        latency = now_ms() - t0
        subtask_trace["calls"].append({
            "tier": tier, "model": call_meta.model,
            "input_tokens": call_meta.input_tokens,
            "output_tokens": call_meta.output_tokens,
            "latency_ms": latency,
            "cost_estimated": call_meta.cost_estimated,
            "output_chars": len(answer.answer),
            "output_lines": answer.answer.count("\n") + 1,
        })
        verdict = invoke pyramid-verify(node, tier, answer, call_meta, pyramid_config + spent_so_far)
        spent_so_far += verdict.estimated_cost_used
        subtask_trace["tier_path"].append(tier)
        subtask_trace["verdicts"].append(verdict)

        ledger.append({"node": node, "tier": tier,
                       "verifier_confidence": verdict.verifier_confidence,
                       "decision": verdict.decision, "reason": verdict.reason,
                       "cost_used": verdict.estimated_cost_used})

        if verdict.decision == "accept" or tier == pyramid_config.max_tier:
            results[node.id] = answer
            subtask_trace["final_tier"] = tier
            subtask_trace["accepted"] = (verdict.decision == "accept")
            break
        tier = verdict.next_tier
        last_answer = answer
        last_verdict = verdict
    trace["subtasks"].append(subtask_trace)

trace["totals"] = compute_totals(trace, dag)            # see "Totals" below
write_trace(trace, pyramid_config.session_dir)
invoke pyramid-aggregate with (results, ledger, dag, task, trace, pyramid_config)
```

### Capturing call metadata (`call_meta`)

After each `task`-tool dispatch, capture:

- `model`: the model id passed to the call.
- `input_tokens`, `output_tokens`: read from the tool's response if surfaced.
  If not surfaced, estimate as `len(prompt) / 4` and `len(answer_text) / 4`
  respectively, and set `cost_estimated: true`.
- `cost`: `(input_tokens + output_tokens) * multiplier(model)` (multipliers
  defined in `pyramid-verify` SKILL).

### Totals computed at the end

```
totals = {
  "subtasks_total": <int>,
  "subtasks_accepted": <int>,
  "subtasks_max_tier_reached": <int>,
  "tier_dispatches": {1: <int>, 2: <int>, 3: <int>},
  "verifier_calls": <int>,
  "tokens_in_total": <int>,
  "tokens_out_total": <int>,
  "cost_tw_units": <float>,
  "cost_tw_units_always_tier3": <float>,   # sum(tokens) * multiplier(tier3 model)
  "savings_pct": <float>,                   # 1 - cost / cost_always_tier3
  "wall_clock_ms": <int>
}
```

### Trace file location

Write the trace as JSON to:

```
<session_dir>/files/pyramid-runs/<task-slug>-<YYYYMMDD-HHMMSS>.json
```

Where `task-slug` is the first 40 ASCII-safe chars of the task description,
kebab-case. Create the `pyramid-runs/` directory if missing.

If `session_dir` is not provided, fall back to
`./.pyramid-runs/` in the current working directory.

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
- `trace`: the structured trace dict (post-`compute_totals`)
- `pyramid_config`: includes `output_level`, `session_dir`

The aggregate skill produces the user-facing response **and** writes the
human-readable per-run report Markdown file alongside the JSON trace.

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
