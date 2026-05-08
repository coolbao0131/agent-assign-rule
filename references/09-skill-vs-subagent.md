# 09 — Skill vs Sub-Agent: The Meta-Decision

The first decision when adding new functionality to a Claude Code project is "skill or sub-agent?" Get this wrong and you either flood main context or pay sub-agent dispatch overhead for nothing. Read when designing new capability.

## The fundamental difference

| Aspect | Skill | Sub-agent |
|---|---|---|
| **Metaphor** | A cookbook chapter | A hired specialist |
| **Context window** | Shares the parent agent's | Fresh, isolated 200K |
| **Loading** | Auto-loaded by Claude based on `description` | Explicitly invoked via `Agent` tool |
| **Best at** | Knowledge, conventions, formats, transforms | Multi-step work with intermediate state |
| **Returns** | Inline content the parent uses immediately | A summary the parent treats as a result |
| **Cost shape** | Tokens added to parent context | Full model run, then summary returned |
| **State** | Stateless (just instructions) | Stateful within its run (its own conversation) |
| **Composition** | Skill can dispatch sub-agent (`context: fork`) | Sub-agent can preload skill (`skills: [...]`) |

## Decision tree

```
[New capability]
      │
      ▼
Q1. Is this a fixed transformation, format, or template?
    (e.g. "convert PDF to text", "format release notes", "apply the X style guide")
      ├── YES → Skill
      └── NO ↓
      ▼
Q2. Does it need its own multi-step working memory?
    (e.g. "investigate this bug", "review the PR across files", "design this module")
      ├── YES → Sub-agent
      └── NO ↓
      ▼
Q3. Will it produce a lot of intermediate output the parent doesn't need to see?
      ├── YES → Sub-agent (isolation is the value)
      └── NO ↓
      ▼
Q4. Is it knowledge / rules the parent should *apply directly*?
      ├── YES → Skill
      └── NO ↓
      ▼
Default → Skill is usually safer. Sub-agents have dispatch overhead;
          skills inline. If unsure, start with skill, escalate later.
```

## Concrete examples — skill OR sub-agent?

| Capability | Pick | Why |
|---|---|---|
| Convert PDF to markdown | Skill | One-shot transform, deterministic |
| Format git log into release notes | Skill | Template + format |
| Apply the company writing style guide | Skill | Knowledge applied inline |
| Review a 5-file diff for security issues | Sub-agent | Multi-file, isolated context, returns report |
| Investigate why CI is flaking | Sub-agent | Multi-step, exploratory, lots of intermediate reads |
| Design the schema for a new feature | Sub-agent | Reasoning + drafts + revisions, needs own memory |
| Generate a CHANGELOG entry from commits | Skill | Format-based |
| Plan a database migration | Sub-agent | Multi-step impact analysis |
| Lookup of conventions in this codebase | Skill | Knowledge lookup |
| Run the test suite and summarize failures | Sub-agent | Bash work + interpretation |
| The agent-assign-rule meta-skill itself | Skill | Decision rules + templates, applied inline |

## When BOTH make sense (composition)

### Skill that dispatches a sub-agent

`SKILL.md` frontmatter:
```yaml
---
name: deep-research
description: ...
context: fork
---
```

`context: fork` runs the skill's body in an isolated sub-agent context. Use when:
- The skill produces a lot of intermediate output (research notes, draft documents)
- You don't want the skill's working state to pollute the parent
- The result is a single deliverable (report, summary) the parent uses

### Sub-agent that preloads a skill

`.claude/agents/code-reviewer.md` frontmatter:
```yaml
---
name: code-reviewer
description: ...
skills: [security-review, style-guide]
---
```

The sub-agent loads those skills as part of its system prompt at startup. Use when:
- The sub-agent needs domain knowledge as a prerequisite
- Multiple sub-agents share that knowledge (factor it into a skill rather than duplicating)
- The knowledge updates independently from the agent definition

## Cost model

**Skill cost** = description scan (~50-100 tokens, every turn) + body load (when triggered, full size, into parent context)

**Sub-agent cost** = description scan (one-time at registration) + per-dispatch (full prompt + isolated run + summary return)

A skill that's loaded once per session is cheaper than a sub-agent dispatched once per session, IF the skill body is small. A skill body of 10K tokens loaded into the parent costs the same regardless of session length, while a sub-agent's 10K of intermediate state stays in the sub-agent's context and only the summary returns.

**Heuristic:**
- Body < 2K tokens? → Skill
- Body > 5K tokens AND only relevant occasionally? → Sub-agent
- Body 2K-5K, used frequently → Skill
- Body 2K-5K, used rarely → Sub-agent

## Common confusion patterns

### "It's a skill but I want isolation"
That's `context: fork`. You don't need to convert the skill to a sub-agent — fork it.

### "It's a sub-agent but it's just applying rules"
That should be a skill. Sub-agents pay dispatch overhead; if the work has no multi-step memory, you're wasting it.

### "It does both — it has knowledge AND multi-step work"
Pick the bigger axis. If the multi-step work dominates → sub-agent that preloads a knowledge skill. If the knowledge dominates → skill with `context: fork` for the multi-step parts.

### "I want the user to type /foo to invoke it"
That's a skill. Slash commands are a skill primitive. Sub-agents are dispatched programmatically by the orchestrator, not by user invocation.

### "I want the orchestrator to decide whether to use it"
Either works:
- Skill: Claude auto-loads based on `description`
- Sub-agent: orchestrator decides via Agent tool

Same auto-routing surface; different execution shape.

## Anti-patterns

- **Skill that returns a giant report** → should be a sub-agent (isolated context)
- **Sub-agent that just applies a style guide** → should be a skill
- **Skill with `context: fork` for trivial work** → fork overhead exceeds benefit
- **Sub-agent that always asks for the same context** → that context belongs in a skill the sub-agent preloads
- **Skill and sub-agent with overlapping description** → Claude can't pick reliably; one of them isn't doing its job

## Migration paths

If you started with a skill and it's growing:
1. > 5K tokens of body → consider `context: fork`
2. Multi-step work emerging → split: skill (knowledge) + sub-agent (process), with sub-agent preloading the skill
3. Skill description triggering on too many things → that's a meta-skill (like agent-assign-rule); narrow the trigger or split into multiple

If you started with a sub-agent and it's underused:
1. Mostly applying rules → demote to skill
2. Always called with the same prompt → fold that prompt into the sub-agent's system prompt body
3. Always returns the same shape → that shape belongs in the prompt as a template
