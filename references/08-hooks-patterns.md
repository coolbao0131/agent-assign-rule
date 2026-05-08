# 08 — Hooks Patterns for Sub-Agent Orchestration

`PreToolUse` / `PostToolUse` / `PreCompact` hooks are deterministic guards that run shell commands at fixed lifecycle points. They are how you enforce safety, route by size, and self-correct sub-agents without the agent's cooperation.

## Hook lifecycle (relevant to sub-agents)

| Hook | Fires when | Use for |
|---|---|---|
| `UserPromptSubmit` | User sends a message | Prompt validation, redirect to skill |
| `PreToolUse` | Before any tool call | Allowlist check, size-based routing, destructive-pattern blocking |
| `PostToolUse` | After tool call completes | Linter / test feedback loop, output trimming, audit logging |
| `PreCompact` | Before auto-compaction | Capture about-to-be-lost context, abort if budget exceeded |
| `Stop` | Before agent finishes its turn | Cleanup, summary, persistence |
| `SubagentStop` | When a sub-agent finishes | Aggregate sub-agent outputs, validate envelope |

## Hook output schema

Hooks communicate by stdout JSON or exit code:

| Mechanism | Effect |
|---|---|
| `exit 0` | Success, continue |
| `exit 2` | Block the operation, agent sees the stderr |
| stdout `{"hookSpecificOutput": {"permissionDecision": "deny"}}` | Block with structured reason |
| stdout `{"hookSpecificOutput": {"additionalContext": "..."}}` | Inject text into agent's next turn |
| stdout `{"continue": false, "stopReason": "..."}` | Halt the run |

**Critical:** stdout JSON gives finer-grained control than exit codes. Use it when you need the agent to learn from the rejection.

## Pattern 1 — Bash safety gate (PreToolUse)

Block destructive Bash commands before they run.

`.claude/hooks/bash-safety.sh`:
```bash
#!/usr/bin/env bash
# Read JSON from stdin
input=$(cat)
tool=$(echo "$input" | jq -r '.tool_name')

[ "$tool" != "Bash" ] && exit 0

cmd=$(echo "$input" | jq -r '.tool_input.command')

# Block destructive patterns
if echo "$cmd" | grep -qE '(rm -rf [/~]|git push --force|curl[^|]*\| *(ba)?sh)'; then
  cat <<EOF
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Destructive pattern blocked by hook: $cmd"
  }
}
EOF
  exit 0
fi

exit 0
```

Wire in `.claude/settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{"type": "command", "command": ".claude/hooks/bash-safety.sh"}]
      }
    ]
  }
}
```

→ Related: `references/06-tool-allowlist.md` (Bash safety rules), `references/07-failure-recovery.md` Op-1 (permission/allowlist mismatch).

## Pattern 2 — Auto-summarization gate (PostToolUse)

When a sub-agent's tool call returns a giant blob, replace it with a summary so the orchestrator's context doesn't explode.

> **Critical schema note**: `updatedToolOutput` REPLACES the tool's response — it must match the tool's response shape (Bash uses `{stdout, stderr, interrupted, isImage}`; other tools differ). `additionalContext` only APPENDS to the agent's next turn; using it for size reduction would make context worse, not better. Pattern 2 only works with `updatedToolOutput`.

`.claude/hooks/trim-large-output.sh` (Bash-specific — adapt the response shape if you wire it for other tools):
```bash
#!/usr/bin/env bash
input=$(cat)
tool=$(echo "$input" | jq -r '.tool_name')
[ "$tool" != "Bash" ] && exit 0

stdout=$(echo "$input" | jq -r '.tool_response.stdout // ""')
stderr=$(echo "$input" | jq -r '.tool_response.stderr // ""')
size=${#stdout}

if [ "$size" -gt 50000 ]; then
  head=$(echo "$stdout" | head -20)
  tail=$(echo "$stdout" | tail -10)
  trimmed="[Output trimmed: $size chars total. First 20 lines:]
$head

[... ${size} chars elided ...]

[Last 10 lines:]
$tail"

  jq -n --arg out "$trimmed" --arg err "$stderr" \
    '{hookSpecificOutput: {hookEventName: "PostToolUse",
      updatedToolOutput: {stdout: $out, stderr: $err, interrupted: false, isImage: false}}}'
fi

exit 0
```

This is a token-budget defense. Without it, a `Bash` call producing a 200KB log dump pollutes the orchestrator forever after.

→ Related: `references/05-output-contracts.md` (the tool's response shape that `updatedToolOutput` must match).

## Pattern 3 — Auto-routing by file size (PreToolUse, claude-gemini-bridge style)

Intercept large reads and re-route to Gemini.

```bash
#!/usr/bin/env bash
input=$(cat)
tool=$(echo "$input" | jq -r '.tool_name')

[ "$tool" != "Read" ] && [ "$tool" != "Glob" ] && exit 0

paths=$(echo "$input" | jq -r '.tool_input.file_path // .tool_input.pattern')
total_size=$(du -bc $paths 2>/dev/null | tail -1 | awk '{print $1}')

# > 50k tokens (~ 200KB rough) → delegate to Gemini
if [ "$total_size" -gt 200000 ]; then
  result=$(gemini -p "Read $paths and return: 1) file structure, 2) key definitions, 3) any obvious issues. Be concise.")
  jq -n --arg r "$result" \
    '{hookSpecificOutput: {hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Routed to Gemini for large-context reading",
      additionalContext: $r}}'
  exit 0
fi

exit 0
```

This is the heart of `claude-gemini-bridge`. Claude proceeds as if it had read the files itself, but the read tokens never hit Claude's context.

→ Related: `references/01-decision-tree.md` Q4 (large reads → Explore/Gemini), `references/03-cross-cli-delegation.md` (Gemini delegation).

## Pattern 4 — Auto-fix loop (PostToolUse linter feedback)

After every `Edit`, run lint. If it fails, push the error back into the agent's next turn so it self-corrects.

```bash
#!/usr/bin/env bash
input=$(cat)
tool=$(echo "$input" | jq -r '.tool_name')

[ "$tool" != "Edit" ] && [ "$tool" != "Write" ] && exit 0

file=$(echo "$input" | jq -r '.tool_input.file_path')

# Only TS / JS files
case "$file" in *.ts|*.tsx|*.js|*.jsx) ;; *) exit 0 ;; esac

if ! lint_output=$(pnpm lint "$file" 2>&1); then
  jq -n --arg out "$lint_output" \
    '{hookSpecificOutput: {hookEventName: "PostToolUse",
      additionalContext: "Lint errors after your edit:\n\($out)\nFix them in your next action."}}'
fi

exit 0
```

The agent reads `additionalContext` in its next turn and self-corrects without human prompting. **Don't block (exit 2) for lint failures** — that stops the agent dead. Inject the feedback and let it fix itself.

→ Related: `references/07-failure-recovery.md` (retry-with-feedback, self-correction loop).

## Pattern 5 — Token budget gate (PreCompact)

Detect when auto-compaction is about to fire, snapshot the transcript, and halt the run if the orchestrator's own budget tracker has flagged overspend.

> **Critical schema note**: `PreCompact` input fields are `trigger` (`"manual"` for user `/compact`, `"auto"` for context-fill auto-compact), `custom_instructions`, and `transcript_path` (path to a JSONL file — NOT inline transcript). The hook itself cannot count tokens; budget tracking must happen in the orchestrator's dispatch logic, which writes a sentinel file the hook reads.

```bash
#!/usr/bin/env bash
input=$(cat)
trigger=$(echo "$input" | jq -r '.trigger // "manual"')
transcript_path=$(echo "$input" | jq -r '.transcript_path // ""')

# Snapshot the about-to-be-compacted transcript (always — useful for debugging)
if [ -n "$transcript_path" ] && [ -f "$transcript_path" ]; then
  cp "$transcript_path" "/tmp/preCompact-snapshot-$(date +%s).jsonl"
fi

# Halt auto-compact if the orchestrator's budget tracker says we've overspent.
# The orchestrator writes /tmp/agent-budget-exceeded when its running token sum
# exceeds the budget; this hook only enforces the halt.
if [ "$trigger" = "auto" ] && [ -f /tmp/agent-budget-exceeded ]; then
  cat <<EOF
{
  "continue": false,
  "stopReason": "Budget exceeded — pipeline halted before auto-compact. Re-plan with smaller scope."
}
EOF
fi
exit 0
```

`PreCompact` fires on both `/compact` and auto-compact; gate only on `trigger == "auto"` so that user-initiated compacts always succeed. For budget tracking itself, the orchestrator must keep its own running total of dispatch input/output tokens and write the sentinel file when it crosses the threshold — the hook has no way to count.

→ Related: `references/07-failure-recovery.md` Type 5 (token budget exceeded), `references/10-anti-patterns.md` #3 (token amplification).

## Pattern 6 — SubagentStop validator

Every time a sub-agent finishes, validate its output before returning to the orchestrator.

```bash
#!/usr/bin/env bash
input=$(cat)
sub_output=$(echo "$input" | jq -r '.subagent_output')

# Try to parse as JSON envelope
if ! echo "$sub_output" | jq -e '.status' >/dev/null 2>&1; then
  jq -n '{hookSpecificOutput: {hookEventName: "SubagentStop",
    additionalContext: "Sub-agent did not return valid JSON envelope. Re-dispatch with explicit schema."}}'
  exit 0
fi

# Validate required keys
if ! echo "$sub_output" | jq -e '.status and .files and .headline' >/dev/null 2>&1; then
  jq -n '{hookSpecificOutput: {hookEventName: "SubagentStop",
    additionalContext: "Sub-agent envelope missing required keys (status/files/headline)."}}'
fi

exit 0
```

This catches contract violations before the orchestrator chains the next phase.

→ Related: `references/05-output-contracts.md` (JSON envelope schema), `references/07-failure-recovery.md` Type 2 (invalid JSON / wrong shape).

## Composing hooks safely

- **Matching hooks run in parallel, not in registration order.** If multiple hooks register for the same event + matcher, Claude Code runs them concurrently. Do NOT design a chain where one hook's exit gates another's input.
- **Identical hook commands are deduplicated.** Registering the same command twice (e.g. once at user scope and once at project scope) will run it once, not twice.
- **A denying hook does NOT short-circuit the others.** If two `PreToolUse` hooks both fire and one denies, the deny still wins (any deny blocks the tool call), but both hooks executed. Don't put expensive work in a hook expecting "an earlier deny will save me from running".
- **Don't share state between hooks via in-flight signals** — they may run concurrently. Use durable artifacts (sentinel files, log lines) only.
- **Test hooks in isolation** before registering. A broken `PreToolUse` can lock you out of all tool use.
- **Log everything.** Hook bugs are silent. `set -x` in a dev hook, write logs to `~/.claude/hooks.log`.

## When NOT to use a hook

- Behavior the agent itself reliably handles via prompt — don't add hook complexity for what `RULES:` can enforce.
- One-off custom workflows — hooks are global; use a custom sub-agent instead.
- Replacing user judgment for high-risk operations — hooks should defer to human, not pre-approve.

## Hook anti-patterns

| Anti-pattern | Why it's bad |
|---|---|
| `exit 2` on lint failure | Kills the agent instead of letting it self-correct |
| Long-running hook (>3s) | Every tool call pays the latency |
| Hook that calls another LLM | Cost blows up on high-frequency `PreToolUse` |
| Hook that mutates project state | Side effects make agent behavior non-reproducible |
| Universal `Bash` hook with no matcher | Fires on every Bash, slow + brittle |
