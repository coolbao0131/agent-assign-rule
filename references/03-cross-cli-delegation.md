# 03 — Cross-CLI Delegation (Codex / Gemini)

When to hand work to OpenAI Codex CLI or Google Gemini CLI instead of dispatching another Claude sub-agent. Read when Q4 (large context) or Q6 (second opinion) of the decision tree fires.

## When to delegate to Codex CLI

Codex (GPT-5.x / o-series) is preferred for:

1. **Token-efficient mass refactor** — for the same refactor, Codex typically uses ~¼ the tokens of Claude. If the task is "rename X to Y across 200 files" or "convert 80 callsites to the new API", Codex is the cost-efficient pick.
2. **Long autonomous terminal work** — Codex's Terminal-Bench 2.0 score is ~77.3%. It does well on multi-step shell-driven tasks where Claude tends to stop and ask.
3. **Second-opinion / rescue when Claude is stuck** — Claude's bias is to over-engineer; Codex's bias is to take shortcuts. When their conclusions disagree, that's the human-decision moment.
4. **Bottom-up architecture review** — Codex tends to look at structure-first, where Claude tends to be feature-first.

## When NOT to delegate to Codex

- Sub-agent already has full task context — switching CLI loses that context. Stick with Claude.
- Task requires reading repo conventions / CLAUDE.md style — Claude has lower friction here since it's the same model family writing them.
- High-risk auth / payment / crypto code — keep Claude for the final pass; use Codex only for exploratory diagnosis.

## Codex CLI invocation patterns

The most common patterns (per `claude-multiagent-boiler-plate`):

```bash
# Read-only diagnosis (high reasoning effort)
codex exec -s read-only -m gpt-5.3-codex -c reasoning_effort=xhigh \
  "Investigate why the auth tests fail intermittently. Read tests/auth/, src/auth/. Output: root cause + 3 candidate fixes." \
  -o /tmp/diagnosis.txt

# Workspace-write (full auto, sandboxed)
codex exec -s workspace-write --full-auto \
  "Apply the fix from /tmp/diagnosis.txt option 2. Run tests after. If tests fail, revert and report."
```

**Flag glossary:**
- `-s read-only` / `-s workspace-write` — sandbox mode. Prefer `read-only` for diagnosis, `workspace-write` for edits. Same enforcement-boundary concept as Claude sub-agent tool allowlists — see `references/06-tool-allowlist.md`.
- `-m <model>` — pin a specific Codex model. Aliases vary by Codex CLI version.
- `-c reasoning_effort=xhigh` — boost for hard problems. Default is fine for routine work.
- `--full-auto` — no human-in-loop prompts. Pair with `-s workspace-write` and trust your sandbox.
- `-o <path>` — write structured output to a file the orchestrator can read back.

## The AGENT_FAILED takeover pattern

For robust fallback when Codex chains fail, use a wrapper that tries multiple models in order, then surrenders explicitly:

```bash
codex_with_fallback() {
  local prompt="$1"
  for model in gpt-5.3-codex o4-mini gpt-4.1-mini; do
    if codex exec -s read-only -m "$model" "$prompt" -o /tmp/out.txt; then
      cat /tmp/out.txt
      return 0
    fi
  done
  echo "AGENT_FAILED"
  return 1
}
```

The Claude orchestrator should detect the literal `AGENT_FAILED` string and announce takeover:

> ⚡ Claude takeover — Codex agents failed, Claude taking over.

Then dispatch a Claude sub-agent (or do it directly) for the same task.

## When to delegate to Gemini CLI

Gemini's win conditions:

1. **Large-context whole-repo / monorepo analysis** — Gemini 2.5 Pro supports 1M-2M tokens. The `claude-gemini-bridge` uses an 800k-token safety cap.
2. **High-volume free reads** — Gemini CLI gives ~1,000 free model requests/day. Use it for the read-heavy phase of a workflow.
3. **Big log analysis** — JSON / CSV / log dumps that would blow Claude's context.
4. **50+ file comparison** — diffing large sets of files for patterns.

## Gemini CLI invocation patterns

Direct invocation:

```bash
gemini -p "Read all files under src/ matching *.handler.ts. List every endpoint that does NOT validate auth before reading req.body. Output: file:line + handler name."
```

Keep prompts narrow — Gemini works best with explicit scope, not "review the codebase".

## The claude-gemini-bridge auto-routing pattern

Instead of manual `gemini -p`, install `claude-gemini-bridge` as a `PreToolUse` hook. It intercepts Claude's tool calls and re-routes when:

- A single tool call would read > **50k tokens**, OR
- The call touches **3+ files**, AND
- Total stays within Gemini's **800k-token safety cap**

The bridge converts the call into a Gemini batch query, runs it, and **substitutes the structured result for the original tool return value**. Claude proceeds as if it had read the files itself, but the read tokens didn't hit Claude's context.

See `references/08-hooks-patterns.md` for the hook wiring.

## Three-way hybrid workflow (Gemini + Codex + Claude)

The classic cost-optimal flow for big tasks:

| Phase | Tool | Why |
|---|---|---|
| 1. Broad reading / log analysis | Gemini ($0) | Free quota, huge context |
| 2. Mass mechanical edits | Codex ($ low) | Token-efficient |
| 3. Architecture / final review | Claude ($ high) | Reasoning depth |

Reported cost reduction: ~60-70% versus Claude-only. See `examples/three-way-hybrid.md`.

## Cost-aware decision rules

- If Claude can do it in < 50k tokens → just use Claude. CLI handoff overhead isn't worth it.
- If the task is read-heavy + reasoning-light → Gemini.
- If the task is write-heavy + mechanical → Codex.
- If the task is reasoning-heavy (architecture, security, novel debugging) → Claude.
- If two CLIs disagree on a high-risk decision → escalate to human.
