# Example — Orchestrator-Worker

The classical pattern: one main agent decomposes the work, dispatches specialized sub-agents, and integrates their outputs. Use when a task has 2-4 distinct skill domains but a single ordering / decision authority.

## Scenario

> "Add a new endpoint `/users/:id/audit-log` that returns the audit history for a user. Update the OpenAPI spec, add the handler with auth, write tests, and document it in the README."

This has 4 sub-tasks across 3 skill areas: implementation (handler + tests), specification (OpenAPI), documentation (README). One ordering matters — handler before docs that describe it.

## Setup

`.claude/agents/`:
- `backend-implementer.md` — Sonnet, full tools, scoped Bash
- `test-writer.md` — Sonnet, full tools, scoped Bash
- `docs-writer.md` — Sonnet, Read/Edit/Write/Grep
- `code-reviewer.md` — Sonnet, Read/Grep/Glob (read-only)

Main agent: Opus. (See `references/02-model-selection.md` for the model-selection rationale.)

## Flow

```
[User] → [Main agent (Opus)]
              │
              │ 1. Plans the work as a markdown file (plan.md)
              │ 2. Dispatches sub-agents in sequence
              │
              ▼
        ┌──── backend-implementer ──── adds handler + auth, returns
        │                              {status, files, exported_types}
        │
        ▼
        ┌──── test-writer ─────────── adds tests against the handler
        │     (gets exported_types)   returns {status, test_files, passing}
        │
        ▼
        ┌──── docs-writer ─────────── updates OpenAPI spec + README
        │     (gets handler signature) returns {status, files, headline}
        │
        ▼
        ┌──── code-reviewer ─────────── final read-only pass on the diff
        │                                returns markdown report
        │
        ▼
[Main agent integrates results, reports to user]
```

## Why orchestrator-worker fits

- **Sequential dependencies**: tests need handler signatures; docs need both.
- **Clear domain split**: each sub-agent has one clear skill area.
- **Single integration point**: main agent reads each result and decides whether to proceed.
- **Audit need**: read-only reviewer at the end, separate from the implementers.

## Concrete dispatch — implementer

```markdown
You are working on: api-gateway
Read first:
- src/api-gateway/CLAUDE.md
- .claude/rules/auth.md
- .claude/rules/api-conventions.md
Path: src/api-gateway/

Commands:
- Build: pnpm build
- Test: pnpm test
- Health: curl localhost:3000/health

TASK: Add GET /users/:id/audit-log endpoint
Returns the user's audit history (paginated, default 50 per page).
Requires authenticated user with `audit:read` scope OR target user
matching the requester (self-read).

Acceptance criteria:
1. Handler at src/api-gateway/handlers/users.audit.handler.ts
2. Route registered with auth middleware (audit:read scope check)
3. Self-read allowed when req.user.id === req.params.id
4. Pagination via ?page=&limit= (limit max 100)
5. Returns {data: AuditEntry[], pagination: {page, limit, total}}

RULES:
- Auth check BEFORE any DB query (no IDOR via 404 vs 403 timing)
- Use the existing AuditService, don't query the table directly
- Return 403 (not 404) when authorized user accesses another user's log without scope
- Output: JSON {"status", "files", "exported_types": ["GetAuditLogResponse"], "headline"}
```

## Concrete dispatch — test-writer (chained)

```markdown
You are working on: api-gateway tests for /users/:id/audit-log
Read first:
- src/api-gateway/handlers/users.audit.handler.ts (just created)
- src/api-gateway/CLAUDE.md
- existing tests in src/api-gateway/handlers/__tests__/

The implementer returned this envelope (verbatim):
<paste implementer's JSON here>

TASK: Write tests for /users/:id/audit-log
Covers happy path, auth boundaries, pagination, errors.

Acceptance criteria:
1. Self-read returns 200 with own audit log
2. Authenticated user without scope reading another user → 403
3. Unauthenticated request → 401
4. Pagination: limit=10 returns at most 10
5. Limit > 100 → 400 (validation)
6. All tests pass

RULES:
- Use existing test fixtures in __tests__/fixtures/
- Don't introduce a new mocking library
- Don't write tests against private functions; only the handler's HTTP surface
- Output: JSON {"status", "test_files", "tests_added", "tests_passing", "headline"}
```

## Anti-patterns to avoid in this pattern

1. **Parallelizing the chain.** The handler must exist before tests can import its types. Don't parallelize.
2. **Forgetting to forward the upstream output.** Test-writer needs the handler signature; if you don't paste implementer's envelope, it'll guess.
3. **Letting reviewer have Edit.** Reviewer's value is the read-only constraint. Tools: `Read, Grep, Glob` only.
4. **Skipping the reviewer.** It's tempting on a "small" addition. Don't skip it for code that touches auth.
5. **Token amplification.** Each sub-agent should return a JSON envelope ≤ 500 tokens, not a full report. The main agent re-reads files if it needs detail.

## When this pattern is wrong

- Single-domain task → just dispatch one sub-agent (or do it directly)
- No sequential dependency → consider parallel exploration instead
- Task is exploratory / no clear plan upfront → use a single general-purpose sub-agent first to scope it, then orchestrate
- Task is small enough for a single sub-agent to handle the whole thing → orchestrator overhead isn't worth it
