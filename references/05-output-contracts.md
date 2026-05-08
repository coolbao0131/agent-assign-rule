# 05 — Output Contracts

How sub-agents should return results so the orchestrator can act on them. Read when designing a multi-phase pipeline or when an orchestrator can't reliably parse what its sub-agent returned.

## Three output shapes

| Shape | When to use | Parser |
|---|---|---|
| **Prose** | Single-shot informational answer the orchestrator will summarize for the user | Read-and-paraphrase |
| **Structured markdown** | Single-shot review, audit, or report the user (or another agent) consumes directly | Section headers |
| **JSON envelope** | Multi-phase pipeline; output feeds the next phase as input | Strict schema parse |

## When prose is OK

The sub-agent's output goes directly to the user, the orchestrator only relays it, and there's no downstream programmatic step.

Examples:
- "Explain why the X module uses Y pattern."
- "Summarize the auth flow."
- Built-in `Explore` results.

If the orchestrator needs to chain another sub-agent based on this output, prose is the wrong choice.

## When structured markdown wins

Use when:
- The output has predictable sections (Summary / Issues / Notes)
- A human reads the output, not just code
- Section headers are stable enough to grep / split on

The code-reviewer template uses this. The orchestrator can:
- Show the report directly to the user
- Grep for `^## Blocking issues` to count blocking issues
- Grep for `⚠️ Escalate` to detect human-in-loop signals

Required: the prompt's `RULES:` section must specify exact section headers — `Summary`, `Blocking issues`, `Non-blocking issues`, `Notes`. Don't say "use whatever headers make sense"; the parser has no fallback.

The reviewer's tool allowlist (see `references/06-tool-allowlist.md`) is what makes this output contract enforceable — without `Read, Grep, Glob` only, the reviewer can edit instead of report and the markdown contract becomes meaningless.

## When JSON wins (and how to enforce it)

Use a JSON envelope when:
- Phase N's output is Phase N+1's input
- The orchestrator must decide control flow based on result
- You need to count, sum, or compare across multiple sub-agent runs

⚠️ Claude Code does NOT support a `structured_output` field in `.claude/agents/*.md` frontmatter (the GitHub feature request was closed as not planned). Enforcement happens entirely via prompt body.

### The standard JSON envelope

```json
{
  "status": "complete" | "failed" | "partial",
  "reason": "<one-line summary; required if status != complete>",
  "files": ["path/to/changed.ts"],
  "headline": "<one-line summary for orchestrator log>",
  "<task-specific keys>": ...
}
```

Every dispatch should require these four:
- `status` — pipeline control
- `reason` — required when not `complete`, optional otherwise
- `files` — what got touched (so orchestrator can re-read or pass to reviewer)
- `headline` — for orchestrator log; **NOT a full report**

Task-specific keys go after. Example for a backend implementer:

```json
{
  "status": "complete",
  "files": ["src/services/user-service/agent.schema.ts", "agents.service.ts"],
  "headline": "Added lastActive field, indexed; PATCH updates it; tested",
  "types_added": ["lastActive: Date"],
  "tests_added": ["agents.service.spec.ts: PATCH updates lastActive"]
}
```

### Embedding JSON in the prompt

```markdown
RULES:
- ...
- Output: JSON only, in this exact shape, with no surrounding prose:

  {
    "status": "complete" | "failed" | "partial",
    "reason": "<required if not complete>",
    "files": [...],
    "headline": "<one line>",
    "types_added": [...],
    "tests_added": [...]
  }

  If you cannot complete the task, return status="failed" with a reason.
  Do NOT return free-form prose explaining what you did.
```

Sub-agents return prose by default. The "JSON only, no surrounding prose" hint matters; without it you get prose + JSON code block + more prose.

## Validation checklist (orchestrator side)

Before chaining the next phase, the orchestrator must verify:

1. ✅ Output is valid JSON (try parse; on failure, treat as `status: "failed"`)
2. ✅ `status` is one of the allowed values
3. ✅ If `status != "complete"`, `reason` is present and non-empty
4. ✅ Required task-specific keys are present
5. ✅ Types match the schema (string vs array, etc.)
6. ✅ Referenced files actually exist (run `Read` or `Test -f`)

Any failure → halt the pipeline. **Never silent-fail.** A bad sub-agent output that gets passed to the next phase corrupts the whole run.

## Schema-first delegation

For long pipelines, define the contract before writing any sub-agent:

```yaml
# .claude/contracts/backend-implementer.yaml
inputs:
  - task: string
  - acceptance_criteria: array of strings
outputs:
  - status: enum [complete, failed, partial]
  - files: array of file paths
  - types_added: array of strings  # for SDK chain
validation:
  - all files exist
  - tests pass
  - swagger spec parses
```

Then the sub-agent's prompt cites the contract: "Your output MUST match `.claude/contracts/backend-implementer.yaml`."

This is heavyweight — only do it for pipelines you'll run repeatedly.

## Ground Truth Injection — verifying claims

The big trap: trusting the sub-agent's prose claims about its own work.

> "I found 23 unused fields and removed them all."

The orchestrator must NOT believe this. Add a deterministic verification step:

1. Run a `code-reviewer` sub-agent (Read-only) over the diff to confirm count
2. OR run a script: `grep -c "removed" /tmp/diff.log`
3. OR run the test suite and check pass count

If verification disagrees with the sub-agent's claim, mark the run failed and escalate. This is the difference between a pipeline that ships bugs and one that doesn't.

See `references/07-failure-recovery.md` for retry-with-feedback.

## Common output-contract bugs

| Bug | Symptom | Fix |
|---|---|---|
| Sub-agent returns prose + JSON | Parser fails | Prompt: "JSON only, no surrounding prose" |
| Missing `status` field | Orchestrator can't decide flow | Make `status` first key in schema |
| `status: "complete"` but tests didn't run | False success | Add ground-truth verification phase |
| `headline` is 2 paragraphs long | Orchestrator log floods | Prompt: "headline is ONE line, ≤ 80 chars" |
| Different sub-agents use different schemas | Orchestrator branches everywhere | Define one envelope; all sub-agents share it |
