# 02 — Model Selection (Opus / Sonnet / Haiku)

> Syntax verified against current Claude Code docs at writing time. Aliases (`opus`, `sonnet`, `haiku`) auto-track each tier's latest model and are preferred. Full model IDs vary by Claude Code version — run `/agents` or check current docs before pinning.

How to pick the right Claude model for a given role, and how to wire it in `.claude/agents/*.md` frontmatter.

## The three models at a glance

| Alias | Strengths | Cost / Speed | Default role |
|---|---|---|---|
| `opus` | Top-tier reasoning, architecture design, complex debug; large default context | Slowest, most expensive | Orchestrator + final reviewer + high-risk implementer |
| `sonnet` | Balanced speed + quality, "workhorse" coder | Mid | Standard implementer, layer-specific sub-agents |
| `haiku` | Very fast, very cheap, low latency | Fastest, cheapest | Read-only explorers, classifiers, lightweight workers |

## Selection criteria

Decide by asking, in order:

1. **Will this make a high-risk or irreversible decision?** (architecture, auth, schema, security)
   → **Opus**. Don't optimize cost on the wrong axis.
2. **Is this read-only / search / classification / extraction?**
   → **Haiku**. Fast and cheap is the whole point.
3. **Is this standard implementation — write code, fix bug, refactor a module?**
   → **Sonnet**. Default workhorse.
4. **Does it need long, multi-step reasoning over a large surface?**
   → **Opus** if reasoning depth dominates; **Sonnet** if breadth dominates.
5. **Is it called by another agent and only returns a summary?**
   → Match to the **inner work**, not the outer caller. A Haiku-returning-paths is fine even if the main agent is Opus.

## The classic cost-saving pattern

> **Main agent in Opus (commander). Sub-agents in Sonnet/Haiku (workers).**

This is the single biggest cost lever. Reasoning tokens stay where they matter (architecture decisions, plan synthesis, final review); cheap models handle the bulk file reads / writes.

Counterexample — when sub-agents should ALSO be Opus:
- The sub-agent makes its own architectural decisions (e.g. a sub-agent designing a new module's API)
- The sub-agent reviews high-risk code (auth, payment, crypto)
- The sub-agent debugs a non-obvious cross-layer race condition

## Per-persona recommendations

| Persona | Model | Why |
|---|---|---|
| `code-reviewer` | Sonnet | Pattern-matching against best practices; doesn't need deep reasoning unless flagged high-risk |
| `code-reviewer-security` | Opus | Threat modeling needs reasoning depth |
| `codebase-explorer` | Haiku | Read + summarize; speed matters more than judgment |
| `debugger` | Opus | Root cause analysis is reasoning-heavy |
| `test-writer` | Sonnet | Mechanical; only Opus if the SUT is unusual |
| `migration-planner` | Opus | High-risk impact mapping |
| `security-auditor` | Opus | Reasoning about adversarial conditions |
| `data-scientist` | Sonnet | Pandas / sklearn workflows are well-trodden |
| `lint-fixer` | Haiku | Per-rule mechanical fixes |
| `pr-summarizer` | Haiku | Diff → bullets |

## Frontmatter syntax

In `.claude/agents/<name>.md`:

```yaml
---
name: code-reviewer
description: ...
tools: Read, Grep, Glob
model: sonnet
---
```

Allowed values for `model`:
- **Aliases (preferred):** `opus`, `sonnet`, `haiku` — auto-track each tier's latest model
- **Full model IDs:** version-specific strings like `claude-<tier>-<version>` — only use when reproducibility requires pinning. Run `/agents` to see what your CLI version exposes.
- `inherit` — match the parent agent's model
- Omitted — same as `inherit`

**Always prefer aliases in templates.** Full IDs rot with each release. Pin only when a specific incident requires bit-for-bit reproducibility, and document the reason next to the pin.

## Cost / quality tradeoffs to internalize

- **Opus reasoning is ~5× Sonnet cost per token. Sonnet is ~5× Haiku.** Order of magnitude different — don't pick by aesthetic preference.
- **Opus has an over-spawning tendency.** It opens sub-agents reflexively; pair it with hard rules in the orchestrator's system prompt: "Do not delegate tasks under 3 steps."
- **Haiku is good at structure, weak at semantics.** It can find files, list functions, classify by keyword — but don't ask it to judge whether code is "well-designed".
- **Sub-agent token amplification means Sonnet workers + Opus orchestrator can still cost more than expected.** Keep sub-agent return summaries < 500 tokens.

## When to override the default

Default is "main = Opus, workers = Sonnet/Haiku". Override when:

- **Trivial main task, no orchestration needed** → main = Sonnet directly, no sub-agents
- **Bulk parallel work, no reasoning per item** → main = Sonnet (orchestration is light), workers = Haiku
- **Single critical decision, no parallelism** → main = Opus, no sub-agents (don't pay sub-agent dispatch overhead)

## Quick frontmatter library

```yaml
# Reviewer (read-only)
model: sonnet
tools: Read, Grep, Glob

# Explorer (fast read-only)
model: haiku
tools: Read, Grep, Glob

# Implementer (general)
model: sonnet
tools: Read, Edit, Write, Grep, Glob, Bash

# Debugger (root cause + fix)
model: opus
tools: Read, Edit, Write, Grep, Glob, Bash

# Security auditor (read-only, deep)
model: opus
tools: Read, Grep, Glob

# Data scientist (analysis, write scripts)
model: sonnet
tools: Bash, Read, Edit, Write
```

## Worked example — picking models for a refactor pipeline

Task: rename `deprecated_field` to `new_field` across 200 files; tests + types follow.

| Phase | Work | Model | Why |
|---|---|---|---|
| 1. Plan / impact mapping | Cross-layer reasoning, identify breaking changes, sequence the rollout | **Opus** | High-risk multi-service decision; reasoning depth matters |
| 2. Mass mechanical rename | 200 files, low-judgment edits | **Codex CLI** (or Sonnet workers) | ~¼ the token cost on mechanical work; Sonnet workers OK if Codex unavailable |
| 3. Verifier | Run tests, diff against original API, count rename sites | **Sonnet** | Standard implementer work — execute commands, interpret output |
| 4. Final review | Catch missed sites, inconsistent renames; check auth-adjacent files | **Sonnet** code-reviewer + **Opus** on auth-adjacent files | Default Sonnet review; escalate to Opus only where Q2 fires |

The reasoning-heavy phases stay in Opus; mechanical bulk goes to Codex; verification is mid-tier. **Don't put Phase 2 in Opus** — paying Opus prices for mechanical edits is the most common cost mistake on refactors.

Compare with the cross-layer worked example in `references/01-decision-tree.md` which is shape-driven (which sub-agent dispatches when); this one is model-driven (which model for which kind of cognitive work).
