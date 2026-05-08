# 10 — Anti-Patterns

The five most common mistakes when designing sub-agent dispatch. Each is paired with a symptom (how to recognize you're in it) and a fix (concrete way out).

## 1. Over-spawning

**Symptom:** Five sub-agent dispatches for a 3-step task. Most return < 200 tokens of output. Orchestrator spends more time dispatching than doing.

**Cause:** Opus has a documented bias to delegate reflexively. Without a hard rule, it opens sub-agents for anything that "could be a separate concern".

**Fix:**
- In the orchestrator's system prompt: "Do not delegate tasks under 3 distinct steps. Handle short tasks directly."
- Use the Q1 of the decision tree as a check: if it's a one-line edit on a single file, just do it.
- Audit: count sub-agent dispatches per session. If > 5 for a non-pipeline task, you're over-spawning.

## 2. Lossy hand-off

**Symptom:** Sub-agent's first action is asking a clarifying question. Or worse — it doesn't ask, it guesses, and the guess is wrong.

**Cause:** Dispatch prompt was 1-3 lines. Sub-agent has no view of the parent conversation.

**Fix:**
- Always pack the five-section template: Context / Commands / Goal / Acceptance / Rules. See `references/04-prompt-template.md`.
- For chained sub-agents, EXPLICITLY include upstream output in the downstream prompt:
  ```
  Previous step's output (passed verbatim):
  <Phase 1 JSON>
  ```
- Repeat the 2-3 most critical rules in `RULES:` even if they're in CLAUDE.md. Sub-agents skip past line 280 of long rule files.

## 3. Token amplification (the "20K bloat" trap)

**Symptom:** Orchestrator's context grew from 8K to 38K after one orchestrator-worker run. Cost scaling worse than expected.

**Cause:** Sub-agents return 2K-token summaries. Ten sub-agents × 2K = 20K bloat in the orchestrator forever after. Every subsequent orchestrator turn pays this in input tokens.

**Fix:**
- Output contracts must specify length. Force JSON envelopes with `headline` ≤ 80 chars, not free-form summaries.
- Sub-agents return **paths and headlines, not content**. The orchestrator can re-read specific files if needed; it shouldn't be carrying the sub-agent's full report in context.
- For long-running sessions with many sub-agent dispatches, dispatch a `summarizer` sub-agent periodically to compact the orchestrator's history.

## 4. Bash blanket-allow

**Symptom:** Sub-agent has `tools: ..., Bash` and no `permissions.allow` restrictions. Eventually a confused instance runs `rm -rf` or `git push --force`.

**Cause:** Convenience during development; never tightened before deployment.

**Fix:**
- ALWAYS restrict via `.claude/settings.json` `permissions.allow`. Specific commands or argument patterns only.
- Pair with a `PreToolUse` hook for destructive-pattern blocking. See `references/08-hooks-patterns.md` Pattern 1.
- Audit: `grep -r "Bash" .claude/agents/` and verify every entry has a corresponding allowlist or hook guard.

## 5. Trusting prose claims

**Symptom:** Sub-agent says "I removed 23 unused fields and all tests pass." You believe it. Two days later production breaks because (a) it actually removed 21, (b) tests didn't run, or (c) the claim was hallucinated.

**Cause:** Treating sub-agent output as ground truth instead of a claim to verify.

**Fix:**
- Ground Truth Injection: every quantitative or success claim gets a deterministic verification step. See `references/07-failure-recovery.md`.
- Run the test command yourself, check the exit code. Never rely on "tests pass" in prose.
- Diff the actual changes (`git diff --stat`) and compare to the sub-agent's claim.

## 6. Mixing skill and sub-agent confusedly

**Symptom:** A skill that produces 5K tokens of output every time it loads. Or a sub-agent that's just applying a style guide.

**Cause:** Picking by familiarity rather than by `references/09-skill-vs-subagent.md`'s decision tree.

**Fix:**
- Skill that produces lots of intermediate output → add `context: fork` (run in isolated sub-agent context).
- Sub-agent that's just applying rules → demote to skill.
- Body > 5K tokens, used rarely → sub-agent.
- Body < 2K tokens → skill.

## 7. The 8+ fan-out trap

**Symptom:** 12 sub-agents dispatched in parallel. Coordination time exceeds benefit. Some sub-agents return inconsistent shapes. Aggregation phase becomes a debugging exercise.

**Cause:** "More parallel = more speed" intuition; ignored the cap.

**Fix:**
- Default fan-out: 4 sub-agents.
- Cap: 8 (research-backed upper bound).
- If you genuinely have > 8 independent items, batch them: 8 sub-agents each handling 4 items > 32 sub-agents handling 1 each.
- Aggregate phase requires a fixed envelope; loose schemas amplify with parallelism.

## 8. Override-blind dispatch

**Symptom:** High-risk schema migration was dispatched to a Sonnet sub-agent because the read-phase was big. Skipped Opus + reviewer because Q4 (large reads) fired first in the pre-fix decision flow.

**Cause:** Treating decision tree as "first match wins" without recognizing Q2 (high-risk) is an override.

**Fix:**
- Read `references/01-decision-tree.md` carefully — Q2 sets constraints, then Q3-Q6 picks the pattern under those constraints.
- High-risk + large reads = orchestrator-worker (or external read delegation) UNDER Opus + reviewer.
- Q2's constraints don't disappear because Q4 also matched.

## 9. Over-defensive prompts

**Symptom:** Sub-agent prompt is 8K tokens of "if X happens do Y... if Y happens do Z...". Sub-agent spends most of its budget reading the prompt.

**Cause:** Trying to handle every possible failure inline rather than using hooks / verification phase.

**Fix:**
- Prompt covers the happy path + 2-3 most likely failure modes.
- Edge cases handled by `PreToolUse` / `PostToolUse` hooks (deterministic, cheap).
- Verification handled by a downstream `verifier` sub-agent.
- Prompt < 6K tokens; if larger, the task is too big for one sub-agent — split.

## 10. Description trigger sprawl

**Symptom:** Skill or sub-agent fires on prompts it has no business handling. User says "explain this function" and the meta-skill kicks in.

**Cause:** `description` field is too broad — claims to handle adjacent topics.

**Fix:**
- Description names specific trigger phrases, not topics.
- Include explicit "DO NOT trigger for" exclusions for adjacent skills.
- Test triggering: ask "does this prompt match my description's trigger language?" If yes but the prompt isn't this skill's job, narrow the description.
- Avoid "use proactively before any X" — that's a tax on every X.

## Quick reference — symptoms → which anti-pattern

| Symptom | Likely anti-pattern |
|---|---|
| 5+ sub-agents dispatched for small task | 1 (over-spawning) |
| Sub-agent asks clarifying questions | 2 (lossy hand-off) |
| Orchestrator context jumped 20K+ after a run | 3 (token amplification) |
| `rm -rf` or `git push --force` ran unexpectedly | 4 (Bash blanket-allow) |
| Production broke despite "tests pass" claim | 5 (trusting prose) |
| Skill outputs 5K tokens every load | 6 (skill/sub-agent confusion) |
| Parallel dispatch returns inconsistent shapes | 7 (8+ fan-out) |
| High-risk task got cheap-and-fast treatment | 8 (override-blind) |
| Sub-agent spends budget reading prompt | 9 (over-defensive prompts) |
| Skill fires on irrelevant prompts | 10 (description sprawl) |
