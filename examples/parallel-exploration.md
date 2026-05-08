# Example — Parallel Exploration

Multiple sub-agents searching different regions concurrently. Use when you have N independent areas to investigate and the orchestrator's only job is aggregation.

## Scenario

> "We're starting a security audit of the auth subsystem. Map the attack surface across the gateway, the auth service, the session store, and the SDK clients."

Four independent areas. No ordering between them. Each produces a structured map. Aggregator stitches them.

## Setup

`.claude/agents/codebase-explorer.md` (Haiku, read-only) — see `agent-templates/codebase-explorer.md`.

Main agent: Sonnet (parallelism is the savings; main doesn't need Opus for aggregation).

## Flow

```
[Main agent (Sonnet)]
       │
       ├──┐
       │  ├── codebase-explorer #1: gateway
       │  ├── codebase-explorer #2: auth-service
       │  ├── codebase-explorer #3: session-store
       │  └── codebase-explorer #4: sdk-clients
       │
       ▼
   (all 4 dispatched in a single message — true parallelism)
       │
       ▼
   Each returns JSON: {status, results: [{path, role, key_symbols}]}
       │
       ▼
[Main aggregates → unified attack-surface map]
```

## Concrete dispatches (all in one message)

The main agent sends ONE message with FOUR Agent tool calls. Claude Code executes them concurrently.

### Dispatch 1
```markdown
You are codebase-explorer for the gateway.
Path: src/gateway/

TASK: Map the auth-relevant attack surface in the gateway.
Find:
- Every entry point that processes auth-related requests (login, logout, token refresh, MFA)
- Every middleware in the auth chain (order matters)
- Where session cookies / tokens are read/written
- Where rate limiting applies (or doesn't) to auth endpoints

Output: JSON {status, request_type: "map", results: [...], headline}
Hard limit: ≤10 files in results.
```

### Dispatch 2
```markdown
You are codebase-explorer for auth-service.
Path: src/auth-service/

TASK: Map the auth-relevant attack surface in auth-service.
Find:
- Token issuance, validation, revocation paths
- Password handling (storage, comparison, reset)
- MFA enrollment & verification
- Cross-service auth contracts (what the gateway / SDK rely on)

Output: JSON {status, request_type: "map", results: [...], headline}
Hard limit: ≤10 files in results.
```

### Dispatches 3, 4 — same shape for session-store and sdk-clients

## Aggregation in main agent

After all four return, main agent:

1. Validates each envelope — `status: "complete"` and `results` non-empty.
2. Builds a unified map:
   ```
   ## Attack surface map (auth subsystem)
   ### Gateway (path: src/gateway/)
   - <file:role>
   ### Auth-service (path: src/auth-service/)
   - <file:role>
   ...
   ```
3. Spots cross-component patterns ("session-store and gateway both write the cookie — divergence risk").
4. Hands the map to a `security-auditor` sub-agent for the actual audit.

## Why parallel-exploration fits

- **Independent areas** — no sub-agent needs another's output.
- **Same shape** — all four return the same `results: [...]` JSON, so aggregation is mechanical.
- **Read-heavy** — reading 4 codebases sequentially would saturate the main agent's context; parallel sub-agents keep each ≤200K and only return summaries.
- **Speed matters** — security audit context-gathering is the bottleneck for the human reviewer.

## Cap and defaults

- **Default fan-out: 4** (heuristic safe starting point)
- **Cap: 8** (research-backed upper bound where coordination overhead exceeds parallelism benefit)

If you genuinely have > 8 areas, batch them: 4 sub-agents each handling 2 areas, not 8 sub-agents handling 1 each.

## Anti-patterns

1. **Different output shapes per sub-agent.** Aggregation becomes a programming exercise. Force the same JSON envelope on all.
2. **Allowing sub-agents to read each other's areas.** That's not parallel — that's chaos. Pin each sub-agent to its `Path:` strictly.
3. **Aggregating with Opus.** Aggregation is mechanical merging; Sonnet is enough.
4. **Returning full file contents.** Sub-agents return paths + one-line roles. Main re-reads if needed.
5. **Fan-out > 8.** Coordination tax exceeds benefit. Batch or split into rounds.
6. **Mixing fan-out with sequential work.** If sub-agent #4's output feeds sub-agent #5, that's not parallel — restructure.

## When this pattern is wrong

- Sequential dependencies between tasks → use orchestrator-worker
- Single area to investigate → use one sub-agent, no fan-out
- Tasks that need different skills (one debug, one review, one refactor) → orchestrator-worker, not fan-out
- Areas overlap significantly → you'll get duplicate findings, restructure the partition

## Cost note

Parallel fan-out uses N× the per-call cost compared to sequential, but N× less wall-clock time. For interactive sessions, the wall-clock saving is usually worth it. For unattended pipelines (CI), sequential is often cheaper since wall-clock doesn't matter.
