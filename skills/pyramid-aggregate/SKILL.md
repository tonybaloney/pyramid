---
name: pyramid-aggregate
description: Merge Pyramid MoA subtask results into a single user-facing response with a cost ledger showing tier escalations and estimated savings.
---

# pyramid-aggregate

Final step of the Pyramid MoA pipeline. Combines per-subtask results into a
single coherent answer and prints the cost ledger so the user can see what
the router did and what it saved.

## Inputs

- `results`: map of subtask id → `pyramid-result` block
- `ledger`: list of `{subtask_id, tier, verifier_confidence, decision, reason, estimated_cost_used}`
- `dag`: original `pyramid-dag` block
- `task`: original user task description
- `spent_so_far`: total estimated cost (relative units)

## Step 1 — Compose the final answer

Walk `dag.nodes` in topological order. For each node, take the corresponding
entry from `results` and:

- **`read` / `verify`**: include the answer text as a brief bullet.
- **`transform` / `write` / `synthesize`**: if the answer describes file edits
  (paths + diffs), summarize them as a "Files changed" block. If it includes
  free-text reasoning, condense to 1–3 sentences.

Produce a single final response with three sections:

1. **Result** — the synthesized answer to the user's original task.
2. **Files changed** — bulleted list of files written/edited across all subtasks
   (deduplicated). Skip if none.
3. **Pyramid MoA ledger** — formatted as below.

## Step 2 — Format the ledger

Render as a markdown table:

```
| Subtask           | Final tier | Confidence | Decision path                  | Cost |
|-------------------|------------|------------|--------------------------------|------|
| <id (kind)>       | T1/T2/T3   | 0.XX       | T1→accept  /  T1→T2→accept     | X.X  |
| ...               | ...        | ...        | ...                            | ...  |
| **Total**         |            |            |                                | X.X  |
```

The `Decision path` column reconstructs the escalation chain by walking the
ledger entries for that subtask in order.

Below the table, print a one-line **savings summary**:

```
Pyramid MoA: total cost = X.X units; always-tier-3 cost would have been Y.Y units → saved (Y.Y − X.X) / Y.Y * 100 = Z%.
```

Where `always-tier-3 cost = sum(cost_of_tier3_for_track) for each subtask`.
Also note any subtasks that hit `max_tier` without reaching the threshold,
prefixed with `⚠`.

## Step 3 — Sanity checks

Before returning:

- Every subtask in `dag.nodes` must appear in both `results` and the ledger.
  If any are missing, add a `⚠ Subtask <id> did not complete` line to the
  ledger and surface this prominently above the result.
- If the response mentions file paths, verify each exists (`view` or `glob`)
  and warn in the ledger if any do not.

## Output

Print the final response (Result + Files changed + Ledger) directly. Do not
return a fenced block — this is the user-facing output, not data for another
skill.
