# pyramid — Pyramid Mixture-of-Agents plugin for Copilot CLI

A Copilot CLI plugin that brings **hierarchical Mixture-of-Agents** routing to
your terminal. Hands a task to the cheapest tier first, verifies the answer,
and only escalates to a more expensive tier when a Hansen-Zilberstein
**Value-of-Computation** rule says it's worth it.

Based on Khaled, *Pyramid MoA: A Probabilistic Framework for Cost-Optimized
Anytime Inference*, [arXiv:2602.19509](https://arxiv.org/abs/2602.19509).

> On MBPP, the router intercepts 81.6% of bugs; on GSM8K/MMLU the system
> nearly matches the 68.1% Oracle baseline while achieving up to 42.9%
> compute savings.

## Install

From GitHub (recommended):

```
copilot plugin install tonybaloney/pyramid
```

From a local clone:

```
git clone https://github.com/tonybaloney/pyramid.git
copilot plugin install ./pyramid
```

Verify:

```
copilot plugin list
```

## Usage

In an interactive Copilot CLI session:

```
/pyramid:pyramid <task description> [flags]
```

Copilot CLI namespaces plugin commands as `/<plugin>:<command>`, so the slash
command for this plugin is `/pyramid:pyramid` (not `/pyramid`).

### Flags

| Flag         | Default          | Description                                                                |
|--------------|------------------|----------------------------------------------------------------------------|
| `--track`    | `claude`         | Model family: `claude`, `gpt`, or `mixed`.                                 |
| `--theta`    | `0.75,0.85`      | Confidence thresholds θ₁,θ₂ for escalating from tier 1→2 and 2→3.          |
| `--max-tier` | `3`              | Hard ceiling on escalation.                                                |
| `--anchor`   | `off`            | If `on`, pass the rejected lower-tier answer to higher tiers as context.   |
| `--budget`   | (none)           | Optional cost cap (relative units) — stops further escalation when hit.    |

### Tier mapping

| Tier | `claude`             | `gpt`            | `mixed`              |
|------|----------------------|------------------|----------------------|
| 1    | `claude-haiku-4.5`   | `gpt-5-mini`     | `claude-haiku-4.5`   |
| 2    | `claude-sonnet-4.6`  | `gpt-5.4`        | `gpt-5.4`            |
| 3    | `claude-opus-4.7`    | `gpt-5.3-codex`  | `claude-opus-4.7`    |
| V    | `claude-haiku-4.5`   | `gpt-5-mini`     | `claude-haiku-4.5`   |

### Example

```
/pyramid:pyramid Find the function that handles password reset and add rate limiting (max 5/hour per user)
```

The router will typically:

1. Decompose into ~3 subtasks: *find function* (read, prior 0.10), *design
   rate-limit approach* (synthesize, prior 0.55), *implement + test*
   (transform+verify, prior 0.40).
2. Run the *find* subtask at tier-1 → verifier confidence ≈ 0.95 → accept.
3. Run *design* at tier-1 → verifier may flag concurrency edge cases →
   confidence 0.6 → escalate to tier-2 → accept.
4. Run *implement* at tier-1 → accept if tests pass.
5. Print the cost ledger so you can see exactly which subtasks needed
   escalation and the estimated savings vs. always running at tier-3.

## How it works

```
/pyramid:pyramid <task>
   │
   ▼
pyramid-orchestrate skill ──────────────────────────────────────────────┐
   │                                                                    │
   │ 1. pyramid-decompose       → DAG of subtasks (id, kind, prior)     │
   │                                                                    │
   │ 2. for each subtask:                                               │
   │      ┌──────────────────────────────────────────────────┐          │
   │      │ tier-N solver (task tool, model=tier-N)          │          │
   │      └──────────────────────────────────────────────────┘          │
   │                          │                                         │
   │                          ▼                                         │
   │      ┌──────────────────────────────────────────────────┐          │
   │      │ pyramid-verify  →  pyramid-verifier (Haiku)      │          │
   │      │  → Hansen-Zilberstein VoC: escalate iff           │          │
   │      │    expected_gain × value > cost_ratio             │          │
   │      └──────────────────────────────────────────────────┘          │
   │                          │                                         │
   │            accept ◀──────┴──────▶ escalate to tier N+1            │
   │                                                                    │
   │ 3. pyramid-aggregate       → final answer + cost ledger            │
   └────────────────────────────────────────────────────────────────────┘
```

## Anchoring (paper §4)

Passing a rejected lower-tier answer to a higher tier sounds helpful but the
paper finds:

- **+19.2pp** accuracy when the lower-tier reasoning is correct.
- **−18.0pp** accuracy when it is incorrect.

Since the only reason we escalated is that we *don't trust* the lower-tier
output, the default `--anchor=off` shows higher tiers only the subtask + the
verifier's critique. Use `--anchor=on` only when you have evidence the lower
tier is usually directionally right.

No MCP server, no hooks, no external runtime — everything piggybacks on the
CLI's built-in `task` tool and its per-call `model` override.

## Citation

```
@article{khaled2026pyramidmoa,
  title  = {Pyramid MoA: A Probabilistic Framework for Cost-Optimized Anytime Inference},
  author = {Khaled, Arindam},
  journal= {arXiv preprint arXiv:2602.19509},
  year   = {2026}
}
```

## License

MIT.
