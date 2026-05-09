---
name: agent-assign-rule
description: Decide how to delegate engineering work — which sub-agent (or none), which model (Opus / Sonnet / Haiku), or which external CLI (Codex / Gemini) — and scaffold the dispatch prompt, tool allowlist, and output contract. Use when the user asks "should I open a sub-agent", "Opus or Sonnet for an engineering task", "how do I split this work", "design an agent for X", or invokes /agent-assign-rule. Also use proactively before dispatching multiple agents/models, or when delegation strategy is ambiguous, high-risk, cross-layer implementation work, or spans multiple services. Do not use for a single straightforward Explore pass, routine one-agent implementation, or pure research (deep-research owns that).
---

# agent-assign-rule

Meta-skill for delegation decisions. It does **not do** the work itself — it tells Claude **whom to give the work to** and **how to brief them**.

## When to trigger

ANY of these activate the skill:

- "should I open a sub-agent for X" / "該不該開 sub-agent"
- "which model — Opus, Sonnet, Haiku" / "Opus 還是 Sonnet"
- "delegate to Codex / Gemini?" / "丟給 Codex / Gemini 嗎"
- "design a sub-agent for ..." / "幫我寫一個 ... agent"
- "split this work" / "怎麼拆"
- explicit `/agent-assign-rule`

DO NOT trigger for:
- One-shot edits to a known file (just do it, no delegation needed)
- Pure research questions (use `/deep-research` instead)
- Already-decided dispatch with prompt drafted (just call `Agent` directly)

## Quick decision flow (read this first, every time)

Evaluate in order. **Q1 is terminal.** **Q2 sets mandatory constraints — apply them, then continue evaluating Q3-Q6 to choose the dispatch pattern under those constraints.** For Q3-Q6, the first match wins.

**Q1.** One-line edit on a single file you can locate?
→ just do it. **STOP.**

**Q2.** High-risk / irreversible? (auth, payment, schema mutation, > 4 services touched, public API change)
→ **CONSTRAINTS:** primary in Opus, mandatory `code-reviewer` handoff, never parallelize, stage as separate PRs.
Apply these, then continue to Q3.

**Q3.** Cross-layer (≥ 2 architectural layers — e.g. schema + API + UI)?
→ orchestrator-worker, sequential dispatch (DRY onion). See `examples/plan-then-execute.md`.

**Q4.** > 50k tokens of intermediate output?
→ `Explore` (built-in, Haiku) for structure-only; Gemini CLI for whole-content reasoning.

**Q5.** Lots of similar small tasks (lint each file, summarize each PR)?
→ parallel fan-out. **Default 4** (heuristic starting point); **cap 8** (research-backed upper bound where coordination overhead exceeds benefit). See `examples/parallel-exploration.md`.

**Q6.** Need a second opinion or stuck on a known Claude weakness?
→ Codex CLI rescue. See `references/03-cross-cli-delegation.md`.

**Default** → handle directly, no delegation overhead.
*(If Q2 constraints are active, "directly" means sequential Opus primary + mandatory reviewer — not unguarded solo execution.)*

## Step-by-step workflow (when user asks "design an agent" / "how do I split this")

### Step 1 — Classify the task
Bucket into one: **explore / implement / review / refactor / debug / migrate / audit**. The bucket determines which `agent-templates/*.md` to use as starting point.

### Step 2 — Estimate context size & cross-cutting scope
- Single service, single file → no sub-agent
- Multi-file but single layer → consider parallel sub-agents
- Cross-layer → orchestrator-worker with sequential dependency map
- > 50k tokens reading → Gemini delegation via hooks (`references/08-hooks-patterns.md`)

### Step 3 — Choose the executor
Decision order:

1. **Built-in subagent** (`Explore`, `Plan`, `general-purpose`) if it fits → cheapest
2. **Custom Claude sub-agent** (`.claude/agents/*.md`) if reusable persona needed
3. **External CLI** (Codex / Gemini) if Claude is wrong tool — see `references/03-cross-cli-delegation.md`

### Step 4 — Pick the model
Read `references/02-model-selection.md`. TL;DR:
- Opus → orchestrator + high-risk implementer + final reviewer
- Sonnet → workhorse implementer
- Haiku → explorer / classifier / lightweight workers

### Step 5 — Draft the dispatch prompt
Use the **five-section template** from `references/04-prompt-template.md`:
Context → Commands → Goal → Acceptance criteria → Rules.

Never send a one-line prompt to a sub-agent. Sub-agents have no view of the parent conversation — pack everything they need.

### Step 6 — Define the output contract
Read `references/05-output-contracts.md`.
- Exploratory work → markdown report
- Multi-phase pipeline → JSON with `status` field, parsed by orchestrator

### Step 7 — Set the tool allowlist
Read `references/06-tool-allowlist.md`. Default: minimum privilege. Reviewer = `Read, Grep, Glob` only. Never `Bash(*)` blanket-allow.

## Hard rules

1. **No `Bash(*)` blanket-allow.** Always restrict via `permissions.allow` or `PreToolUse` hook.
2. **Reviewer agents have no `Edit` / `Write`.** The persona's value depends on read-only enforcement.
3. **Sub-agent prompts are self-contained.** Sub-agents see no parent history — never write `based on what we discussed` or `like the previous one`.
4. **Validate sub-agent output deterministically.** Don't trust prose claims like "I found 23 issues" — run a verification phase (test, grep, schema check) per `references/07-failure-recovery.md`.
5. **Cap parallel fan-out at 8.** Coordination overhead exceeds benefit beyond that.
6. **Cap retry-with-feedback at 3.** Beyond that, escalate to human-in-loop.
7. **structured_output is not supported in sub-agent frontmatter** (closed as not planned). Enforce JSON contracts via prompt body only.

## References (load on demand)

| File | When to read |
|---|---|
| `references/01-decision-tree.md` | Every dispatch decision — the full decision flow with branch examples |
| `references/02-model-selection.md` | Choosing Opus / Sonnet / Haiku for a specific role |
| `references/03-cross-cli-delegation.md` | Considering Codex or Gemini delegation |
| `references/04-prompt-template.md` | Drafting a sub-agent dispatch prompt |
| `references/05-output-contracts.md` | Multi-phase pipelines, schema-first delegation |
| `references/06-tool-allowlist.md` | Configuring `tools:` field for a persona |
| `references/07-failure-recovery.md` | Retry policy, AGENT_FAILED takeover, ground truth injection |
| `references/08-hooks-patterns.md` | PreToolUse / PostToolUse / PreCompact orchestration |
| `references/09-skill-vs-subagent.md` | Meta-decision: should this be a skill or a sub-agent |
| `references/10-anti-patterns.md` | What not to do — over-spawning, lossy hand-off, amplification |

## Agent templates (copy to `<project>/.claude/agents/`)

| Template | Persona |
|---|---|
| `agent-templates/code-reviewer.md` | Read-only PR reviewer (Sonnet) |
| `agent-templates/debugger.md` | Root-cause + fix + verify (Opus) |
| `agent-templates/codebase-explorer.md` | Fast read-only mapper (Haiku) |
| `agent-templates/test-writer.md` | Generates tests against existing code |
| `agent-templates/migration-planner.md` | Schema/API migration impact + sequencing |
| `agent-templates/security-auditor.md` | Read-only vulnerability scan |

## Examples (concrete workflows)

| Example | When |
|---|---|
| `examples/orchestrator-worker.md` | Main agent + N specialized workers (single domain owner per task) |
| `examples/parallel-exploration.md` | 4 explorers fan out across non-overlapping areas |
| `examples/plan-then-execute.md` | Cross-layer (backend → SDK → frontend), strict sequence |
| `examples/code-review-handoff.md` | Implementer → reviewer (Q2 demonstration) |
| `examples/three-way-hybrid.md` | Gemini → Codex → Claude pipeline for cost-optimal mass refactor |
| `examples/iterative-codex-review.md` | 1-3 round adversarial Codex review of a draft design before shipping; includes non-convergence handling and user-escalation protocol |

## Anti-patterns cheatsheet

1. **Over-spawning** — Opus tends to open sub-agents reflexively. If the task is < 3 steps, don't.
2. **Lossy hand-off** — sub-agent gets a 2-line prompt and guesses the rest. Always pack: paths, types from upstream, acceptance criteria, rules.
3. **Token amplification** — 10 sub-agents × 2k summary = 20k bloat in orchestrator forever after. Make sub-agents return paths + headlines, not full reports.
4. **Bash blanket-allow** — `Bash(*)` lets a confused sub-agent run `rm -rf`. Always allowlist.
5. **Trusting prose claims** — "I fixed 12 issues" is unverified. Add a verification phase.
6. **Mixing skill and sub-agent confusedly** — formatting / templates / one-shot transforms = skill. Multi-step delegated work with own context = sub-agent. See `references/09-skill-vs-subagent.md`.
