# Example — Three-Way Hybrid (Claude + Codex + Gemini)

The cost-optimized pipeline: cheap models do bulk work, expensive models make decisions. Reported ~60-70% cost reduction versus Claude-only on suitable tasks.

## Scenario

> "Production has been throwing intermittent 500s on the checkout endpoint. We have ~2GB of access logs from the past week. Find the pattern, propose a fix, and validate it before we deploy."

This task has three phases with very different cost profiles:
- **Phase 1 (read-heavy, reasoning-light)**: scan 2GB of logs for patterns
- **Phase 2 (write-heavy, mechanical)**: implement the candidate fix across N call-sites
- **Phase 3 (reasoning-heavy)**: validate the fix is correct, doesn't introduce regressions

## Pipeline shape

| Phase | Tool | Model | Why | Cost |
|---|---|---|---|---|
| 1. Log triage | Gemini CLI | Gemini 2.5 Pro | Free quota, 1M-2M context | $0 (within free tier) |
| 2. Mechanical fix | Codex CLI | gpt-5.3-codex | ~¼ token cost vs Claude on refactors | Low |
| 3. Validation + deploy decision | Claude (Opus) | Opus | Reasoning depth for "did this fix the issue" | High |

## Flow

```
[User describes the problem to Claude (Opus orchestrator)]
       │
       │ Opus plans the pipeline:
       │  - Phase 1: Gemini for log analysis
       │  - Phase 2: Codex for mechanical fix
       │  - Phase 3: Opus self-review + deploy gate
       │
       ▼
   ┌─── Phase 1: Gemini CLI ────────────────────────────────────┐
   │   gemini -p "Read logs/access.log* (gz). Find requests     │
   │   that ended in 500 on /checkout. Group by:                │
   │     - User-Agent prefix                                    │
   │     - Time-of-day pattern                                  │
   │     - Request size                                         │
   │     - Upstream service correlation                         │
   │   Output: top-5 patterns with sample request IDs."         │
   │                                                             │
   │   Returns: structured analysis (markdown table)             │
   └────────────────────────────────────────────────────────────┘
       │
       │ Opus reads Gemini's analysis. Hypothesizes:
       │ "Pattern 1 (large bodies on slow connections + 30s timeout)
       │  → buffering issue in nginx config? Or app-side timeout?"
       │
       │ Opus dispatches a small Claude sub-agent (Sonnet) to
       │ confirm hypothesis by reading the relevant config + handler.
       │
       ▼
   ┌─── (verification dispatch — small) ─────────────────────────┐
   │   Sonnet sub-agent reads:                                   │
   │   - nginx config                                            │
   │   - checkout handler code                                   │
   │   - upstream service timeouts                               │
   │   Returns: "confirmed — handler timeout 25s, nginx 30s,     │
   │   upstream 35s. The mismatch causes the 500 pattern."       │
   └────────────────────────────────────────────────────────────┘
       │
       │ Opus identifies the fix: align timeouts.
       │ Multiple files need updating (nginx config + 3 service files).
       │ This is mechanical — perfect for Codex.
       │
       ▼
   ┌─── Phase 2: Codex CLI ──────────────────────────────────────┐
   │   codex exec -s workspace-write --full-auto                 │
   │     "Align checkout timeouts:                                │
   │      - nginx.conf: proxy_read_timeout 35s                   │
   │      - checkout-handler: timeout 30s                        │
   │      - upstream client: timeout 35s                         │
   │      Verify with: pnpm test, then output diff summary."      │
   │                                                              │
   │   Returns: applied diff + test pass/fail                    │
   └────────────────────────────────────────────────────────────┘
       │
       │ Opus reads Codex's output. If tests pass, proceed to
       │ validation. If tests fail, retry with adjusted scope OR
       │ trigger AGENT_FAILED takeover.
       │
       ▼
   ┌─── Phase 3: Claude (Opus) ──────────────────────────────────┐
   │   Opus dispatches code-reviewer (Sonnet, read-only) to       │
   │   review the Codex-produced diff.                            │
   │                                                              │
   │   Then dispatches security-auditor if checkout is            │
   │   payment-adjacent.                                          │
   │                                                              │
   │   Then synthesizes: deploy / hold / escalate decision.       │
   │   Mandatory human-in-loop for "deploy".                      │
   └────────────────────────────────────────────────────────────┘
       │
       ▼
[User reviews and approves deploy, or asks for adjustments]
```

## Why three-way wins here

- **Phase 1** would cost ~$50+ in Claude tokens for 2GB of logs. Gemini does it on free quota.
- **Phase 2** is N file edits with mechanical changes. Codex does it for ~¼ the Claude token cost.
- **Phase 3** is "did we actually fix it, did we break anything?" — reasoning that pays off in Claude.

Cost split: Phase 1 = $0, Phase 2 ≈ $1, Phase 3 ≈ $5. Claude-only would be ~$15-20.

## Decision rules — when to reach for each CLI

**Pull in Gemini when:**
- Single read > 50k tokens
- Multi-file pattern matching across 50+ files
- Log / JSON / CSV bulk analysis
- You need broad context, narrow output

**Pull in Codex when:**
- Mass mechanical refactor (rename, API migration, framework upgrade)
- Long autonomous shell-driven work (multi-step build / test / fix loops)
- You're stuck — fresh perspective on a known Claude weakness
- Token budget pressure on a non-reasoning task

**Stay with Claude when:**
- Architecture / design decisions
- Anything security or auth adjacent (final pass)
- Novel debugging where reasoning depth matters
- Cross-CLI integration (orchestration is Claude's home turf)

## AGENT_FAILED takeover in this flow

If Phase 2 (Codex) fails:

```bash
# Wrapper used by Opus
codex_with_fallback() {
  for model in gpt-5.3-codex o4-mini gpt-4.1-mini; do
    if codex exec -s workspace-write -m "$model" "$1" -o /tmp/out.txt 2>/dev/null; then
      cat /tmp/out.txt; return 0
    fi
  done
  echo "AGENT_FAILED"; return 1
}
```

Opus detects `AGENT_FAILED` → announces:

> ⚡ Claude takeover — Codex agents failed, Claude taking over the timeout-alignment edits.

Then dispatches a Claude sub-agent (Sonnet) for the same edits. The pipeline continues; no manual intervention needed.

## Anti-patterns

1. **Using Gemini for the reasoning phase.** Phase 3 needs reasoning depth; downgrading to Gemini saves cost but loses correctness — the wrong axis to optimize.
2. **Using Claude for the log scan.** Phase 1's 2GB of logs blow Claude's context. Even with summarization, the cost is wasteful.
3. **Skipping the verification dispatch between Gemini and Codex.** Gemini's analysis is a hypothesis; Codex shouldn't act on it without Claude confirming the hypothesis matches the code.
4. **No human gate before deploy.** Even with all three CLIs lined up, production deploy of a payment-adjacent fix needs human approval.
5. **Silent CLI fallback.** If Codex fails, announce takeover. Silent fallback hides systematic CLI breakage.

## When this pattern is wrong

- Task fits comfortably in Claude alone (< 50k tokens) → don't pay handoff overhead
- Task is single-domain (just reasoning, or just bulk reads) → use one tool, not three
- Latency-sensitive (interactive flow) → CLI handoffs add seconds; serial Claude is faster for small tasks
- No human at the end to approve deploy → don't fully automate critical-path mutations

## Cost monitoring

Three-way is only cheaper if you actually monitor it. Track per-phase tokens:
- Gemini quota usage (1000 free requests/day)
- Codex tokens billed (visible in Codex CLI logs)
- Claude tokens billed (orchestrator's own usage)

If Phase 3 (Claude) tokens grow disproportionately, Codex is leaving too much detail in its summary and Claude is re-reading. Tighten Codex's output contract.
