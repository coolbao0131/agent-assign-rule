# 04 — Sub-Agent Dispatch Prompt Template

The five-section template that turns a vague delegation into a working sub-agent run. Sub-agents see no parent conversation history — every dispatch must be self-contained.

## The five sections

```markdown
You are working on: <service-or-component>
Read first:                        ← (1) CONTEXT
- <path/to/CLAUDE.md>
- <path/to/relevant-rule.md>
Path: <work-directory>

Commands:                          ← (2) COMMANDS
- Build: <build command>
- Test: <test command>
- Lint: <lint command>
- Health: <smoke check>

TASK: <one-line goal>              ← (3) GOAL
<2-3 sentence elaboration. What does success look like in plain words?>

Acceptance criteria:               ← (4) CONSTRAINTS
1. <Concrete, verifiable criterion>
2. <...>
3. <...>
4. <...>

RULES:                             ← (5) RULES / OUTPUT FORMAT
- <Critical do/don't, repeated even if also in CLAUDE.md>
- <...>
- Output: <prose | markdown report | JSON with status field — see ref 05>
```

## Why each section exists

**(1) Context** — sub-agents have no view of the parent agent's reads. State the working directory, point at the project's CLAUDE.md and any relevant rule files explicitly. The sub-agent will Read these as its first action.

**(2) Commands** — without these, the sub-agent will guess. A sub-agent guessing `pnpm test` when the project uses `npm test` wastes a turn discovering it.

**(3) Goal** — one line so the sub-agent's working memory holds it. The 2-3 sentence elaboration is for ambiguity resolution, not detail.

**(4) Acceptance criteria** — 4-5 verifiable items max. These are how the sub-agent self-checks before reporting "done". If you can't verify it, don't list it.

**(5) Rules** — the 2-3 most critical rules, restated even if they're in CLAUDE.md. Sub-agents notoriously skip past line 280 of a long rule file. If a rule MUST be followed, repeat it here.

## Length budget

- Single layer task → ~3000 tokens of prompt
- Cross-layer task → ~4300 tokens per layer sub-agent
- If you're > 6000 tokens of prompt, the task is too big — split it.

If sub-agent gets a 100-token prompt, it asks 5 clarifying questions before working. That's the symptom of an under-packed dispatch.

## Complete example — backend implementer

```markdown
You are working on: user-service
Read first:
- src/services/user-service/CLAUDE.md
- .claude/rules/backend-services.md
- .claude/rules/api-contracts.md
Path: src/services/user-service/

Commands:
- Build: pnpm build
- Test: pnpm test
- Lint: pnpm lint
- Health: curl localhost:8002/health

TASK: Add lastActive field to agent schema
Add a `lastActive` field (Date type) to the Agent schema. Update it
automatically whenever an agent is modified via PATCH /agents/:id.
Field must surface in the Swagger spec.

Acceptance criteria:
1. Field exists on schema with type Date, optional, indexed
2. findOneAndUpdate in agents.service.ts sets lastActive: new Date()
3. Field appears in Swagger via @ApiPropertyOptional
4. Unit test: PATCH an agent, assert lastActive within 1s of now

RULES:
- Use findByIdAndUpdate, never doc.save() (race condition risk)
- Always scope queries by workspaceId (cross-tenant leak prevention)
- Return data directly from controller (ResponseInterceptor wraps)
- Output: JSON {"status": "complete"|"failed", "files": [...], "types_added": [...], "tests_added": [...]} so the orchestrator can chain to the SDK update step.
```

## Complete example — code reviewer

```markdown
You are working on: <service-name> code review
Read first:
- The diff: git diff main...HEAD --stat (then per-file as needed)
- src/<service>/CLAUDE.md
- .claude/rules/code-review.md

Commands:
- (None — reviewer cannot run commands. If you need test results,
   note "test coverage not verified" in your report and let the
   orchestrator dispatch a separate test-runner.)

TASK: Review the changes on this branch against main
Find correctness, security, performance, maintainability, and convention
issues. You have NO Edit/Write/Bash — this is a strictly read-only review.

Acceptance criteria:
1. Every finding has file:line and severity (critical|high|medium|low)
2. No invented findings — if there are zero issues, say so
3. Output uses the exact section headers: Summary, Blocking issues, Non-blocking issues, Notes
4. Escalate auth/payment/crypto findings with "⚠️ Escalate to human"

RULES:
- You have no Edit, Write, or Bash. Do not propose code blocks for application; do not run lint/tests yourself.
- Don't comment on unchanged code unless it directly affects the diff's correctness.
- Distinguish facts from opinions — opinions go in Notes, not Blocking issues.
- Output: markdown report (orchestrator parses by section headers).
```

## Common mistakes (and the fix)

| Mistake | Symptom | Fix |
|---|---|---|
| 100-token prompt | Sub-agent asks 5 questions back | Pack all 5 sections |
| No `Read first` | Sub-agent ignores CLAUDE.md rules | Always list rule files explicitly |
| No `Commands` | Sub-agent guesses build/test commands | Always list them |
| Vague acceptance criteria | Sub-agent claims "done" when it isn't | Use verifiable criteria only |
| Rules buried in CLAUDE.md only | Sub-agent skips past them | Repeat the critical 2-3 in `RULES:` |
| No output format | Orchestrator can't parse result | Always specify prose vs markdown vs JSON |

## When to skip the template

- Built-in `Explore` sub-agent calls — they have a fixed protocol; just write a clear prompt.
- One-off `general-purpose` dispatch for a tiny isolated task — a 4-section variant (drop Commands) is fine.

But never write a one-line dispatch prompt to a custom sub-agent for non-trivial work. That's the #1 cause of bad sub-agent output.
