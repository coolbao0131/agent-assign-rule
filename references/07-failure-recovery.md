# 07 — Failure Recovery and Defensive Orchestration

What to do when a sub-agent returns garbage, times out, runs out of tokens, or claims success without doing the work. Read when designing any pipeline that runs unattended.

This doc covers two layers of failure:
- **Model failures** (Types 1-5 below) — the sub-agent itself misbehaves.
- **Operational failures** (Op-1 to Op-3 below) — the orchestration around the sub-agent breaks. These are usually MORE frequent in practice than model failures.

## Failure modes by type

### Type 1 — Sub-agent times out
Symptom: dispatch returns no result within budget.
Action:
- **Don't kill the pipeline.** Treat as `status: "failed", reason: "timeout"`.
- Capture whatever partial work made it to disk (the sub-agent may have committed files before timing out).
- Decide: retry with smaller scope, escalate, or abandon this branch.

### Type 2 — Sub-agent returns invalid JSON / wrong shape
Symptom: parser throws, required keys missing.
Action:
- Treat as `status: "failed", reason: "schema_violation"`.
- Retry ONCE with the prompt augmented: "Your previous output failed schema validation. Required keys: [...]. Return JSON only."
- If second attempt fails, escalate.

### Type 3 — Sub-agent returns garbage prose ("I'd need more context")
Symptom: prose claiming to need information that should have been in the prompt.
Cause: prompt was under-packed.
Action:
- Don't retry blindly — re-pack the prompt with the missing context the sub-agent named.
- If sub-agent asks for context that's already in the prompt, that's a model failure — switch model (Sonnet → Opus) or escalate.

### Type 4 — Sub-agent claims success but didn't do the work
Symptom: `status: "complete"` but tests still fail / files unchanged.
Cause: hallucinated success.
Action:
- This is why **Ground Truth Injection** exists. Run a deterministic verification phase.
- If verification disagrees, mark failed, dispatch a different model, AND log the model that hallucinated for future avoidance.

### Type 5 — Token budget exceeded
Symptom: sub-agent context fills before completing.
Action:
- The sub-agent's `description` was too broad, or its `Read first` list too long.
- Don't retry as-is — split the task. Two sub-agents reading half each.
- If the task is fundamentally too big for one sub-agent's window, that's a tree-restructure signal — go back to the decision tree and re-plan.

## Operational failure modes

These are what happen when the orchestration AROUND the sub-agent goes wrong. In practice these bite more often than model failures.

### Op-1 — Permission / allowlist mismatch around Bash (most frequent)
Symptom: sub-agent's `Bash` call (test runner, git, package manager) is denied by `permissions.allow`, blocked by a `PreToolUse` hook, or stalls waiting for an interactive permission prompt while the orchestrator thinks the dispatch is still running.

Action:
- The orchestrator must check the sub-agent's return for a `permission_denied` shape and NOT retry blindly — the same call will be denied again.
- Tighten `permissions.allow` proactively before dispatch — see `references/06-tool-allowlist.md` for the standard recipes (reviewer / explorer / test-runner / implementer). If the sub-agent will run `pnpm test`, allow `Bash(pnpm test:*)` ahead of time.
- If denying via `PreToolUse`, return a structured `permissionDecision: "deny"` with a clear reason in `permissionDecisionReason` — the sub-agent can read the reason and adjust, instead of looping.
- For unattended pipelines, set `permissionDecision` to `"allow"` or `"deny"` deterministically; do NOT leave allowed-with-prompt operations on the dispatch path.

### Op-2 — Parallel write collision
Symptom: two parallel sub-agents edit the same file (or files with import dependencies); the second write loses content, or fails because of an unexpected upstream change.

Action:
- Pin parallel sub-agents to non-overlapping `Path:` boundaries — see `examples/parallel-exploration.md`.
- For unavoidable cross-layer work, USE `examples/plan-then-execute.md` (sequential), NOT parallel.
- After detecting a collision (e.g. via `git status` showing unexpected modifications), treat as `status: "failed"` and re-dispatch sequentially. Don't try to merge mid-pipeline.
- Pre-empt by giving each parallel sub-agent's prompt an explicit `Owns: <path-prefix>` line so it understands its territory.

### Op-3 — Lossy / under-packed dispatch prompt
Symptom: sub-agent asks clarifying questions, guesses paths, returns garbage, or claims success without producing real work.

This is the #1 cause of bad sub-agent output, also covered as anti-pattern #2 in `references/10-anti-patterns.md`. It belongs here too because the recovery path matters:

Action:
- Don't retry blindly with the same prompt. Re-pack with the missing context the sub-agent named.
- If the sub-agent named context that WAS in the prompt, that's a model failure — switch model (Sonnet → Opus) or model family (Claude → Codex).
- If the same prompt repeatedly produces clarifying questions across different models, the prompt is the bug — go back to `references/04-prompt-template.md` and pack the five sections explicitly.

### Why these three (and not others)
Frequency-ranked from real Claude Code usage. `Bash` permission mismatches dominate because every test/build/git invocation is a chance to hit them; parallel write collisions are rarer but expensive when they happen; under-packed prompts are common but usually self-correct after one retry. Other operational failures (dirty worktree, MCP server timeout, hook-script syntax errors) are typically downstream of these three or rare enough to handle ad-hoc.

## Retry policy

**Retry budget: 2-3 attempts maximum.** Beyond that, escalate to human.

Retry-with-feedback shape:

```markdown
[Original 5-section prompt]

PREVIOUS ATTEMPT FAILED:
- Reason: <schema violation | timeout | hallucinated success>
- What was wrong: <one-sentence specific>
- This time: <one-sentence concrete fix>

Now retry the task.
```

**Don't loop infinitely on retries.** Each retry costs full input tokens. Three failed retries = ~3× the dispatch cost with no progress. Escalate.

## Human-in-loop escalation

When to require human approval before proceeding:

- ANY operation under Q2 (high-risk) constraints touching production
- Schema mutations / migration mutations
- Auth / payment / crypto code changes
- Two CLIs (Claude vs Codex) disagree on a high-risk decision
- Three retries failed with no progress
- Sub-agent flags `⚠️ Escalate to human` in output

Implementation in Claude Code:
- Use `permissionDecision: "ask"` in hook output to force confirmation
- For SDK use: gate sensitive tools behind `requirePermission: true`
- For interactive use: dispatch the operation as a plan + dry-run, ask the user before executing

## AGENT_FAILED takeover pattern

For external CLI delegation (Codex / Gemini), use a wrapper that explicitly signals fallback:

```bash
codex_with_fallback() {
  local prompt="$1"
  for model in gpt-5.3-codex o4-mini gpt-4.1-mini; do
    if codex exec -s read-only -m "$model" "$prompt" -o /tmp/out.txt 2>/dev/null; then
      cat /tmp/out.txt && return 0
    fi
  done
  echo "AGENT_FAILED"; return 1
}
```

Orchestrator detects literal `AGENT_FAILED` string → announces takeover:

> ⚡ Claude takeover — Codex agents failed, Claude taking over.

Then dispatches a Claude sub-agent with the same prompt body.

The point is **explicit handoff with announcement**, not silent fallback. Silent fallback masks systematic CLI failures; explicit announcement gives the user a chance to stop the pipeline if Codex is broken at a level the takeover won't fix.

## Ground Truth Injection — verifying claims

Sub-agent prose is unreliable. The orchestrator must verify any quantitative or success claim with deterministic code.

Examples:

| Claim | Verification |
|---|---|
| "I removed 23 unused fields" | `git diff --stat` and count removed lines |
| "All tests pass" | Run the test command yourself, check exit code |
| "I added an index" | `grep -c "@Index" src/.../<schema>.ts` |
| "The migration is reversible" | Run forward + backward migration in a sandbox |

**The verification phase is its own sub-agent**, not part of the implementing sub-agent. Dispatch a `verifier` with `Bash, Read, Grep` and a prompt like:

```markdown
TASK: Verify the previous implementer's claims.
Inputs: /tmp/implementer-output.json (claims), git diff main...HEAD (actual changes).
Output: JSON {"verified": true|false, "discrepancies": [...]}
```

If `verified: false`, the orchestrator halts and either retries or escalates.

## Defensive prompt patterns

Things to put in `RULES:` to reduce failure rate:

- **"If you cannot complete the task, return status='failed' with reason. Do not partially complete."** — prevents half-done states the orchestrator can't reason about.
- **"Do not invent file paths. If a file you need does not exist, return status='failed' with reason='missing: <path>'."** — sub-agents fabricate paths under uncertainty.
- **"Output ONLY the JSON envelope. No surrounding prose."** — reduces prose pollution.
- **"After all edits, run [test command]. If tests fail, return status='partial' with the failure log."** — forces self-verification before claiming complete.

## When to abandon, not retry

Stop retrying and re-plan when:

- The same failure mode repeats with different prompts
- The sub-agent's Reads keep returning files that don't exist (planner gave wrong paths)
- Token cost of retries has exceeded the value of the task
- Verification keeps disagreeing — the task may be impossible as specified

A retry loop spending $10 of tokens on a $1 task is a process failure, not a model failure.

## The brittleness budget

Every defensive layer (retry, verification, escalation) costs tokens and time. Spend defenses where they matter:

- **High brittleness (heavy defenses)**: production migrations, security code, paid features
- **Medium brittleness (validation only)**: standard implementation, tests, documentation
- **Low brittleness (trust the sub-agent)**: explorations, rough drafts, prototypes

Don't put a 3-retry verification chain on a doc-comment update. Don't skip verification on a payment code change.
