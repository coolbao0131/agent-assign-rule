# 06 — Tool Allowlist Recipes

Which tools to grant which sub-agent persona. Default is **least privilege** — start with the smallest viable set, expand only when needed.

## Why allowlists matter

Tool access defines what a sub-agent can DO, not just what it CAN'T. A reviewer with `Edit` is no longer a reviewer — it's an implementer that happens to comment. The persona's value depends on its enforced constraints.

Constraints also enable predictability: if the orchestrator knows a sub-agent has `Read, Grep, Glob` only, it knows the sub-agent cannot accidentally write files, so its output can be trusted as observation rather than action.

## Standard recipes

### Reviewer (read-only inspection)
```yaml
tools: Read, Grep, Glob
```
**Why:** No `Edit` / `Write` — the reviewer's value is impartial assessment without modification. No `Bash` — running tests blurs the review/execute boundary. If you need test results, dispatch a separate test-runner sub-agent.

> Implemented in `agent-templates/code-reviewer.md`. See `examples/code-review-handoff.md` for it in action.

### Explorer (fast read-only mapping)
```yaml
tools: Read, Grep, Glob
```
Same as reviewer in shape but Haiku-typed. Used for "where is X defined" / "what files reference Y".

### Test runner (executes, doesn't write code)
```yaml
tools: Bash, Read, Grep
```
**Why:** Bash to run the test command, Read/Grep to interpret output. NO `Edit` — fixing bugs is the implementer's job, not the test runner's.
**Bash restriction:** Limit to test-related commands via `permissions.allow` (e.g. `Bash(pnpm test:*)`, `Bash(pytest:*)`).

### Implementer (general write access)
```yaml
tools: Read, Edit, Write, Grep, Glob, Bash
```
**Why:** Full toolset. **Required:** Bash must be restricted via settings, NEVER `Bash(*)`.

### Debugger (root cause + fix + verify)
```yaml
tools: Read, Edit, Write, Grep, Glob, Bash
```
Same as implementer — debugger needs to investigate, fix, and verify in one sub-agent.

### Security auditor (deep read-only)
```yaml
tools: Read, Grep, Glob
```
Like reviewer, but typically run with Opus for adversarial reasoning depth.

### Data scientist (write scripts, run analysis)
```yaml
tools: Bash, Read, Edit, Write
```
**Why:** Needs to write Python scripts and execute them. No `Glob` because data science usually targets specific known datasets.

### Migration planner (read-only impact analysis)
```yaml
tools: Read, Grep, Glob
```
**Why:** Output is a plan document. Edits happen in a separate implementer phase under Q2 constraints.

## Bash safety — the critical rule

**Never `Bash(*)` blanket-allow.** A confused sub-agent with `Bash(*)` can run `rm -rf`, `git push --force`, `curl ... | sh`, or anything else.

Restrict Bash via `.claude/settings.json` `permissions.allow`:

```json
{
  "permissions": {
    "allow": [
      "Bash(pnpm test:*)",
      "Bash(pnpm build)",
      "Bash(pnpm lint)",
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(git log:*)"
    ],
    "deny": [
      "Bash(git push:*)",
      "Bash(rm -rf:*)",
      "Bash(curl:*)"
    ]
  }
}
```

For sub-agents with broader Bash needs (debugger, implementer), pair with a `PreToolUse` hook that screens for destructive patterns. See `references/08-hooks-patterns.md`.

## MCP tool inclusion

When to add MCP servers to a sub-agent's frontmatter:

- **Add** when the sub-agent needs OAuth-managed external state (Slack, Postgres, Notion, GitHub)
- **Add** when the work is structured and JSON-native (e.g. database queries → MCP postgres rather than Bash psql)
- **Don't add** to the orchestrator if only one sub-agent needs the MCP — scope it to that sub-agent only, to keep the orchestrator's tool descriptions out of every parent turn

Frontmatter syntax:

```yaml
---
name: notion-summarizer
tools: Read, mcp__notion__search, mcp__notion__fetch
model: sonnet
mcpServers:
  - notion
---
```

The token cost of MCP tool descriptions is non-trivial; isolating them to specific sub-agents can save 5-15k tokens of orchestrator context.

## Per-persona quick reference

| Persona | Tools | Notes |
|---|---|---|
| code-reviewer | `Read, Grep, Glob` | No `Bash` — it would blur review/execute |
| codebase-explorer | `Read, Grep, Glob` | Haiku-typed for speed |
| test-runner | `Bash, Read, Grep` | Bash limited to test commands |
| implementer (general) | `Read, Edit, Write, Grep, Glob, Bash` | Bash via allowlist |
| debugger | `Read, Edit, Write, Grep, Glob, Bash` | Bash via allowlist |
| security-auditor | `Read, Grep, Glob` | Opus model |
| data-scientist | `Bash, Read, Edit, Write` | Bash for scripts |
| migration-planner | `Read, Grep, Glob` | Plan only; impl is separate phase |
| pr-summarizer | `Bash, Read` | Bash limited to `git log/diff` |
| lint-fixer | `Read, Edit, Bash` | Bash limited to lint command |

## Anti-patterns

- **Granting `Edit` to a reviewer "just in case"** → it stops being a review
- **`Bash(*)` blanket-allow** → unbounded blast radius on a confused agent
- **Identical tool set on every persona** → defeats the purpose of personas
- **MCP servers on the orchestrator that only one sub-agent needs** → token waste in every parent turn
- **Forgetting to deny destructive Bash patterns** → relying on Claude's good judgment alone is brittle

## Validating your tool config

Before shipping a new agent persona, ask:

1. What's the worst thing a confused instance could do with these tools?
2. Is that reversible?
3. If not, can I split the persona (read-only planner + write implementer with reviewer)?
4. Does Bash need a `permissions.allow` entry?
5. Does any tool exist on this list that the persona has never actually used?

If question 5 is yes, remove the tool. Less is more.
