# agent-assign-rule

A Claude Code **meta-skill** for sub-agent dispatch decisions. It doesn't do the work itself — it tells Claude **whom to give the work to** and **how to brief them**.

## What it answers

- Should I open a sub-agent for this task, or just do it directly?
- Which model — Opus, Sonnet, or Haiku?
- When should I delegate to Codex CLI or Gemini CLI instead of staying with Claude?
- How do I write a dispatch prompt the sub-agent can actually act on?
- What tool allowlist does this persona need?
- What output contract should the sub-agent return?

## Install

Copy the directory into a Claude Code project's skills location, or to global skills:

```bash
# Project-scoped:
cp -r agent_assign_rule/ <your-project>/.claude/skills/agent-assign-rule/

# Or global:
cp -r agent_assign_rule/ ~/.claude/skills/agent-assign-rule/
```

The 6 agent templates can be copied directly into a project's `.claude/agents/` directory:

```bash
cp -r agent-templates/*.md <your-project>/.claude/agents/
```

## Trigger

Claude auto-routes to this skill when you ask:
- "Should I open a sub-agent for X"
- "Opus or Sonnet for an engineering task?"
- "How do I split this work?"
- "Design an agent for X"
- Or invoke `/agent-assign-rule` directly

It does **not** trigger for: routine single-agent work, pure research questions (those go to `/deep-research`), or one-shot edits on a known file.

## Structure

```
agent_assign_rule/
├── SKILL.md                  # Main router: trigger, decision flow, hard rules
├── references/               # Lazy-loaded deep details (per topic)
│   ├── 01-decision-tree.md   # Q1-Q6 + Q2 override mechanic
│   ├── 02-model-selection.md # Opus/Sonnet/Haiku selection
│   ├── 03-cross-cli-delegation.md   # Codex / Gemini delegation
│   ├── 04-prompt-template.md        # 5-section dispatch template
│   ├── 05-output-contracts.md       # prose / markdown / JSON envelope
│   ├── 06-tool-allowlist.md         # per-persona tools
│   ├── 07-failure-recovery.md       # retry / takeover / ground truth
│   ├── 08-hooks-patterns.md         # Pre/PostToolUse + PreCompact
│   ├── 09-skill-vs-subagent.md      # meta-decision
│   └── 10-anti-patterns.md          # 10 common mistakes
├── agent-templates/          # Copy-pasteable to .claude/agents/
│   ├── code-reviewer.md
│   ├── codebase-explorer.md
│   ├── debugger.md
│   ├── migration-planner.md
│   ├── security-auditor.md
│   └── test-writer.md
└── examples/                 # Concrete dispatch workflows
    ├── orchestrator-worker.md
    ├── parallel-exploration.md
    ├── plan-then-execute.md
    ├── code-review-handoff.md
    └── three-way-hybrid.md   # Gemini → Codex → Claude pipeline
```

`SKILL.md` is the entry point — Claude loads it always. The `references/` files are lazy-loaded on demand so the main context stays small.

## Design principles

- **Router + lazy-load**: SKILL.md is compact (~130 lines) and points to references. Same pattern as Anthropic's official skills (pdf, xlsx).
- **Q2 high-risk override**: the decision tree treats high-risk tasks (auth, payment, schema, public API) as constraint-setting, not just one branch — its mandates (Opus + reviewer + no parallel + staged PRs) compose with whichever dispatch pattern Q3-Q6 picks.
- **Aliases over pinned model IDs**: `opus`/`sonnet`/`haiku` track tier-latest; full IDs vary by Claude Code version. Run `/agents` to verify your install.
- **Five-section dispatch prompts**: Context / Commands / Goal / Acceptance / Rules. Sub-agents can't see the parent conversation — every dispatch is self-contained.
- **Output contracts default to JSON envelope** for multi-phase pipelines: `{status, reason, files, headline, ...task-specific}`.
- **Tool allowlists enforce persona constraints**: reviewer = `Read, Grep, Glob` (no Bash, no Edit). Never `Bash(*)` blanket-allow.

## How this was built

The skill went through:
- 2 rounds of NotebookLM deep research (sub-agent patterns + skill-construction details)
- 6 rounds of Codex review (R1-R3 on initial 4-file sample, R1-R3 on full 22-file set)
- 2 graphify passes for cross-reference verification
- 12 concrete fixes catching bugs that survived multiple manual reviews

Notable bugs caught by the review process:
- Hook `additionalContext` doesn't replace tool output (must use `updatedToolOutput` with tool-specific shape)
- `PreCompact` input is `trigger`/`transcript_path`, not inline transcript
- Q2 override mechanic was misordered — high-risk tasks could route to cheap-and-fast branches
- Reviewer prompt example had Bash usage contradicting its no-Bash contract
- Migration-planner asked for file write but had no Write tool

Each fix is documented in the commit history.

## License

MIT
