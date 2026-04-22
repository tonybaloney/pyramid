---
name: pyramid-decompose
description: Decompose a coding task into a DAG of subtasks with per-node difficulty priors. Used by the pyramid-orchestrate skill before tier dispatch.
---

# pyramid-decompose

Produce a JSON DAG of subtasks for the Pyramid MoA orchestrator. The DAG drives
which tier each subtask starts at and what order they run in.

## When to use

This skill is invoked by **`pyramid-orchestrate`** as its first step, after
the user's task has been received via `/pyramid`. Do not invoke it directly
unless you are debugging.

## How to perform decomposition

You have two tools available — use them in order:

### Step 1 — Cheap heuristics (no model call)

Scan the task description for these signals and assemble a draft node list:

| Signal in task                                         | Suggested `kind`     | Prior |
|--------------------------------------------------------|----------------------|-------|
| "read", "show", "find", "list", "search", "grep"       | `read`               | 0.10  |
| "rename", "move", "format", "add import", "typo"       | `transform`          | 0.20  |
| "fix bug", "make test pass", "small change"            | `transform`          | 0.35  |
| "implement", "add feature", "write function"           | `synthesize`         | 0.55  |
| "design", "architect", "refactor across", "optimize"   | `synthesize`         | 0.80  |
| "debug", "why doesn't", "investigate", "root cause"    | `synthesize`         | 0.70  |
| "test", "verify", "check"                              | `verify`             | 0.30  |
| "write file", "create file", "generate"                | `write`              | 0.40  |

Bump prior by +0.10 per ambiguity marker: "somehow", "ideally", "probably",
"figure out", "best way", "you decide". Cap at 0.95.

### Step 2 — Cheap LLM second opinion (Haiku)

Invoke the `task` tool with `agent_type: "general-purpose"`,
`model: "claude-haiku-4.5"`, `mode: "sync"`, and this exact prompt:

```
Decompose this coding task into 1–8 subtasks. For each, return:
- id (kebab-case)
- kind: read | transform | synthesize | verify | write
- description (one sentence, imperative)
- difficulty_prior (0.0–1.0, your honest estimate of how hard this subtask is)
- dependencies (list of ids that must complete first)
- success_criteria (one-sentence, testable)

Output ONLY a JSON array, nothing else.

Task: <user task here>
Heuristic draft: <your draft from step 1>
```

Reconcile the Haiku output with your draft: take the **max** of the two priors
per overlapping subtask, and add any subtasks Haiku identified that you missed.

## Output contract

Return exactly this fenced block to the orchestrator:

````
```pyramid-dag
{
  "task": "<original user task>",
  "nodes": [
    {
      "id": "<kebab-id>",
      "kind": "read|transform|synthesize|verify|write",
      "description": "<one sentence>",
      "difficulty_prior": 0.0-1.0,
      "dependencies": ["<id>", ...],
      "success_criteria": "<one sentence>"
    }
  ]
}
```
````

## Special cases

- If the task is trivially atomic (single read, single one-line edit), return
  exactly one node — do not over-decompose.
- If the task is huge and vague (e.g., "rewrite the whole project"), cap at 8
  nodes and let the higher-tier orchestrator decide whether to recurse.
- DAG must be acyclic. Validate before returning.
