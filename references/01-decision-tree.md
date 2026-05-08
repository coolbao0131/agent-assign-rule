# 01 — Decision Tree

The full decision flow for "should this work be delegated, and to whom?". Read this when SKILL.md's quick flow doesn't give a clean answer.

## How to read this tree

- **Q1 is terminal.** If it matches, stop.
- **Q2 is an OVERRIDE, not a terminator.** If it matches, apply its constraints (Opus primary + mandatory reviewer + no parallelism + staged PRs), then keep evaluating Q3-Q6 to pick the dispatch pattern. Whichever pattern Q3-Q6 selects, it runs under Q2's constraints.
- **Q3-Q6 are evaluated in order; first match wins.**
- **Default fires only if Q3-Q6 all miss.** Under Q2 constraints, Default still means "Opus primary + mandatory reviewer", not unguarded solo execution.

## The decision tree

```
[Incoming task]
      │
      ▼
Q1. One-line edit on a single named file you can already locate?
      ├── YES → Do it directly. No sub-agent. STOP.
      └── NO ↓
      ▼
Q2. High-risk / irreversible?
    (auth, payment, schema mutation, > 4 services touched, public API change)
      ├── YES → APPLY CONSTRAINTS (Opus primary, mandatory code-reviewer handoff,
      │         never parallelize, stage as separate PRs).
      │         Continue to Q3 — the constraints stay active for whichever
      │         dispatch pattern Q3-Q6 picks.
      └── NO ↓ (no constraints active, continue)
      ▼
Q3. Does the task span ≥ 2 architectural layers?
    (e.g. backend schema + SDK types + frontend UI; or DB migration + API + tests)
      ├── YES → Orchestrator-worker with sequential dispatch.
      │         Main agent in Opus. Layer-specific sub-agents in Sonnet.
      │         (See `references/02-model-selection.md` for the rationale.)
      │         See examples/plan-then-execute.md
      └── NO ↓
      ▼
Q4. Will the search / read produce > 50k tokens of intermediate output?
      ├── YES ↓
      │     ├── Need only structure + a few snippets? → Explore (built-in, Haiku)
      │     ├── Need to reason over the whole thing? → Gemini CLI delegation
      │     └── Massive log analysis (free + cheap)?  → Gemini CLI
      └── NO ↓
      ▼
Q5. Lots of similar small tasks (lint each, summarize each, test each)?
      ├── YES → Parallel fan-out in Sonnet/Haiku.
      │         Default 4 (heuristic); cap 8 (research-backed upper bound
      │         where coordination overhead exceeds parallelism benefit).
      │         See examples/parallel-exploration.md
      └── NO ↓
      ▼
Q6. Need a "second opinion" because Claude is stuck or this is a known
    Claude weakness (e.g. mass mechanical refactor, low-level systems)?
      ├── YES → Codex CLI rescue (see references/03-cross-cli-delegation.md)
      └── NO ↓
      ▼
[Default] Handle directly. No delegation overhead.
          (If Q2 constraints are active, "directly" still means sequential
          Opus primary + mandatory reviewer — not unguarded solo execution.)
```

## Worked example — high-risk schema migration with large reads

Task: "Add a `tenant_id` column across user, order, and audit_log tables; backfill 50M rows; update the API and 3 dependent services."

- Q1? No (multi-file).
- Q2? **Yes** — schema mutation + 4+ services + irreversible. **Apply constraints.** Continue.
- Q3? **Yes** — schema + API + 3 services = cross-layer. **Pick orchestrator-worker.**
- Q4? Probably yes for some phases (reading audit_log call sites). Under Q3's plan, dispatch the read-heavy phase to Explore or Gemini, but the resulting plan/edits still go through Q2's reviewer.

Pattern selected: **orchestrator-worker (Q3) under Q2 constraints**.
- Main agent: Opus
- Layer sub-agents: Sonnet
- Read-phase sub-agent: Explore/Gemini for the audit_log scan
- Final: mandatory `code-reviewer` handoff (Q2)
- Rollout: staged PRs (Q2) — additive migration first, then mutation, then services

If Q2 had been NO (e.g. trivial cosmetic change across 2 services), the orchestrator-worker still applies but without the reviewer requirement.

## Branch-specific recipes

### Branch A — Explore (Q4 yes, structure-only)
Use built-in `Explore` subagent. It defaults to Haiku, read-only tools, and is purpose-built for "where is X defined" / "which files reference Y".

Dispatch shape:
```
Agent({
  subagent_type: "Explore",
  description: "Find auth middleware",
  prompt: "Locate all files that define or extend auth middleware. Report file paths and one-line role per file. Search depth: medium."
})
```

Don't use Explore for: code review, design audits, anything past its read window.

### Branch B — Cross-layer orchestration (Q3 yes)
Pattern: **DRY onion**, inside-out.

Example for "add `lastActive` field to user schema, surface in API, show in UI":

1. Main agent (Opus) drafts a Task Plan as a markdown file (`plan.md`).
2. Dispatch `backend-implementer` sub-agent (Sonnet) with full prompt → updates schema, returns `{status, files, types}` JSON.
3. Main agent reads the JSON, injects the new types into the next prompt.
4. Dispatch `sdk-updater` sub-agent → regenerates types.
5. Dispatch `frontend-implementer` sub-agent → uses the type names confirmed in step 4.
6. Main agent reads all three results, dispatches `code-reviewer` sub-agent (Read-only) for final pass.

Critical: **never parallelize cross-layer steps**. Step N's output is step N+1's input.

### Branch C — High-risk (Q2 yes — applies as override)
Add these guardrails on top of whichever pattern fits:
- Main agent in Opus
- Mandatory `code-reviewer` handoff after implementation (read-only tools)
- Stage changes as ≥ 2 PRs (e.g. additive migration first, then mutation)
- Add a verification sub-agent that runs deterministic checks (tests, schema diff, security scan)
- Human-in-loop checkpoint via `permissionDecision: "ask"` before destructive operations

### Branch D — Parallel fan-out (Q5 yes)
- Default 4 concurrent sub-agents (heuristic safe starting point)
- Cap at 8 (research-backed upper bound where coordination overhead exceeds parallelism benefit)
- Use Haiku or Sonnet depending on per-task complexity
- Each sub-agent must return a fixed-shape summary (max ~500 tokens) — not the full work
- Main agent does only aggregation, not re-reading raw output

Anti-pattern to avoid: 10 sub-agents × 2k token summaries = 20k bloat in orchestrator's permanent context.

### Branch E — External CLI delegation (Q4 yes for Gemini, Q6 yes for Codex)
See `references/03-cross-cli-delegation.md`.

## Edge cases not covered above

- **"I just want a second pair of eyes"** (no actual blocker) → use `code-reviewer` sub-agent, not Codex. Cheaper, same model family, same conventions.
- **"The task is small but I want clean isolation"** (e.g. running an experiment that shouldn't pollute main context) → use `general-purpose` sub-agent even if you'd otherwise do it directly. The 200k isolated window is the value, not the model.
- **"It's exploratory but I need to actually fix things found"** → split: Explore sub-agent finds, then main agent fixes (or dispatches debugger). Don't give Explore write access just because it's convenient.

## Smell tests — when you've made the wrong choice

- Sub-agent returns "I'd need more context to help" → you under-packed the prompt. Re-dispatch with full Context + Commands sections (see `04-prompt-template.md`).
- Sub-agent's summary is > 1000 tokens → output contract too loose. Re-dispatch demanding `{status, files, headline}` JSON.
- 5+ sub-agent dispatches in one turn for a small task → over-spawning. Stop, fold the work back into the main agent.
- Orchestrator's context grew by 30k+ after a sub-agent run → token amplification. Sub-agent should have returned paths, not content.
