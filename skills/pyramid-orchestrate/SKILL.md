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
trace_path = write_trace(trace, pyramid_config.session_dir)   # MUST succeed
invoke pyramid-aggregate with (results, ledger, dag, task, trace, trace_path, pyramid_config)
```

### Hard contract — no shortcuts (added in v0.2.1, fixes B1–B4)

These rules are non-negotiable. The host agent may NOT optimize them away
even on tasks it judges "trivial". Smoke benchmark of v0.2.0 found every
one of these contracts broken at least once.

1. **Every DAG node must be dispatched** to `dispatch(tier, node, ctx)`.
   No "resolve inline" shortcut. If a node looks trivial, dispatch it to
   tier 1 anyway — the cost is bounded and the verifier needs a real
   answer to grade. (Defect B4.)
2. **Every dispatched answer must go through `pyramid-verify`.** No
   "verifier round-trip would cost more than the work" shortcut. The
   verifier *is* the safety net; skipping it converts a router into a
   single-tier wrapper. (Defect B3.) If a fast-path is ever justified
   it must be added to `pyramid-verify` itself with a named heuristic
   and a `verifier_skipped: true` trace flag — never decided by the
   orchestrator.
3. **`totals.cost_tw_units` is computed from `subtasks[*].calls[*]`,
   never copied from a per-verdict scalar.** The exact formula is
   pinned in "Totals" below. `verdict.estimated_cost_used` is
   informational only — useful in the per-verdict ledger row, but
   *never* substituted into the totals. (Defect B1.)
4. **`write_trace` runs before `pyramid-aggregate`.** If the JSON write
   fails, raise; do not invoke aggregate. The aggregate skill itself
   re-checks that the JSON file exists before printing the quiet
   summary. (Defect B2.)

### Capturing call metadata (`call_meta`)

After each `task`-tool dispatch, capture:

- `model`: the model id passed to the call.
- `input_tokens`, `output_tokens`: read from the tool's response if surfaced.
  If not surfaced, estimate as follows (B5 fix, v0.2.2):
    - `input_tokens  = max(MIN_TOKENS_PER_CALL, ceil(len(prompt)        / 4))`
    - `output_tokens = max(MIN_TOKENS_PER_CALL, ceil(len(answer_text)   / 4))`
  where `MIN_TOKENS_PER_CALL = 50`. This floor prevents a host-context
  call (where `prompt`/`answer_text` are unavailable and default to "")
  from collapsing tokens — and therefore `cost_tw_units` — to **zero**,
  which silently disables the savings calculation. Set
  `cost_estimated: true` whenever this fallback fires.
- `cost`: `(input_tokens + output_tokens) * multiplier(model)` (multipliers
  defined in `pyramid-verify` SKILL).

If a call genuinely has no LLM dispatch (e.g. a pure inline computation
the orchestrator does itself), do NOT append a `call` entry — leave
`subtask.calls = []`. The B5 floor is for *attempted* dispatches whose
token counts could not be measured, not for things that were never
dispatched.

### Totals computed at the end

`compute_totals(trace, dag)` is the **single source of truth** for cost
numbers. It must always recompute from `subtasks[*].calls[*]`; never
read or sum any per-verdict `estimated_cost_used` field.

```python
MIN_TOKENS_PER_CALL = 50  # B5 floor — see "Capturing call metadata"

def compute_totals(trace, dag, multiplier):
    calls = [c for st in trace["subtasks"] for c in st["calls"]]
    tokens_in  = sum(c["input_tokens"]  for c in calls)
    tokens_out = sum(c["output_tokens"] for c in calls)
    cost_tw    = sum((c["input_tokens"] + c["output_tokens"])
                     * multiplier(c["model"]) for c in calls)
    cost_t3    = (tokens_in + tokens_out) * multiplier(track_models[3])
    # B5: if cost_t3 is 0 (no calls, or all calls had zero tokens despite
    # the floor — should not happen in practice), savings_pct is undefined.
    # Surface that explicitly with `null`, never paper over with 0.
    if cost_t3 > 0:
        savings_pct = round(100 * (1 - cost_tw / cost_t3), 2)
    else:
        savings_pct = None
    return {
      "subtasks_total":             len(trace["subtasks"]),
      "subtasks_accepted":          sum(1 for st in trace["subtasks"] if st["accepted"]),
      "subtasks_max_tier_reached":  sum(1 for st in trace["subtasks"]
                                        if st["final_tier"] == max_tier and not st["accepted"]),
      "tier_dispatches":            {t: sum(1 for c in calls if c["tier"] == t) for t in (1,2,3)},
      "verifier_calls":             sum(len(st["verdicts"]) for st in trace["subtasks"]),
      "tokens_in_total":            tokens_in,
      "tokens_out_total":           tokens_out,
      "cost_tw_units":              round(cost_tw, 2),
      "cost_tw_units_always_tier3": round(cost_t3, 2),
      "savings_pct":                savings_pct,                # None when cost_t3 == 0
      "wall_clock_ms":              sum(c.get("latency_ms") or 0 for c in calls),
      "cost_estimated":             any(c.get("cost_estimated") for c in calls),
    }
```

**Sanity assertion before writing the trace:**
`abs(totals.cost_tw_units - sum((c.in+c.out)*mult(c.model) for c in all_calls)) < 0.01`.
If this fails, the totals are wrong — fix the formula, do not paper over.

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
