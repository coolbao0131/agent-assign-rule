# Example — Plan-then-Execute (DRY Onion)

Cross-layer work where each layer's output is the next layer's input. Inside-out: backend → SDK → frontend, each step producing the contract the next step consumes.

## Scenario

> "Add a `lastActive` field to the User schema, expose it through the SDK, and surface it on the user profile page."

Three layers. Strict ordering. Frontend can't import the type until SDK regenerates; SDK can't regenerate until backend ships the schema.

## Setup

`.claude/agents/`:
- `backend-implementer.md` — Sonnet, full tools
- `sdk-updater.md` — Sonnet, full tools (mostly runs codegen)
- `frontend-implementer.md` — Sonnet, full tools
- `code-reviewer.md` — Sonnet, read-only

Main agent: Opus (planning + integration; high-risk if `lastActive` is used for billing or audit, in which case Q2 constraints apply).

## Flow

```
[Main agent (Opus)]
        │
        │ 1. Drafts plan.md — explicit phase contracts
        │
        ▼
   ┌─── backend-implementer
   │    - Adds field to schema
   │    - Updates service to set it on PATCH
   │    - Adds Swagger annotation
   │    - Writes unit test
   │    - Returns: {status, files, types_added: ["User.lastActive: Date"]}
   │
   ▼
[Main reads result, validates JSON, extracts types_added]
   │
   ▼
   ┌─── sdk-updater
   │    - Runs codegen (gets types from backend's swagger)
   │    - Adjusts any hand-written wrappers
   │    - Returns: {status, files, sdk_version_bumped: "1.4.3"}
   │
   ▼
[Main validates, reads new SDK exports, prepares frontend dispatch]
   │
   ▼
   ┌─── frontend-implementer
   │    - Imports the new SDK type
   │    - Surfaces lastActive on profile page (relative time format)
   │    - Adds component test
   │    - Returns: {status, files, tests_passing}
   │
   ▼
   ┌─── code-reviewer (read-only, full diff)
   │    - Reviews all 3 layers
   │    - Returns markdown report
   │
   ▼
[Main reports to user with the reviewer's findings]
```

## The crucial detail — explicit context bridging

Sub-agents can't see the parent conversation. When dispatching the SDK updater, the main agent must paste the backend implementer's JSON envelope into the SDK dispatch prompt.

```markdown
You are working on: typescript-sdk
Read first:
- packages/sdk/CLAUDE.md
- .claude/rules/sdk-conventions.md

The backend implementer just finished. Their envelope (verbatim):
{
  "status": "complete",
  "files": ["src/services/user-service/user.schema.ts", "user.service.ts"],
  "types_added": ["User.lastActive: Date (optional, indexed)"],
  "swagger_updated": true,
  "headline": "Added User.lastActive; PATCH updates it; tested"
}

TASK: Update the SDK to include the new lastActive field
Run codegen against the latest swagger spec; update wrappers as needed.

Acceptance criteria:
1. Generated User type includes lastActive: Date (optional)
2. SDK version bumped (semver: minor — new field, backward compatible)
3. Existing SDK consumers' tests still pass
4. New unit test exercises the type's optionality

RULES:
- Codegen command: `pnpm sdk:codegen`
- Don't hand-edit generated files; modify generators if needed
- Output: JSON {"status", "files", "sdk_version_bumped", "headline"}
```

Without the backend's envelope pasted in, the SDK updater would have to grep for the change and might miss the type's exact shape (e.g. optional vs required).

## Why plan-then-execute fits

- **Strict sequential dependency** — each phase's output is required input for the next.
- **Different conventions per layer** — each layer has its own CLAUDE.md and conventions; layer-specific sub-agents avoid context bloat.
- **Mistakes are layer-local but expensive to fix late** — finding a wrong type in the frontend after SDK shipped requires reverting two PRs.
- **Reviewer at end is mandatory** — multi-layer changes need a final cross-layer review.

## When Q2 (high-risk) constraints apply

If the field is `lastActive` for general UX → low risk, this flow as-is.

If the field is `accountFrozen` (drives auth decisions) → Q2 active:
- Main agent stays Opus
- Mandatory `code-reviewer` after every phase, not just at the end
- No parallelism (already not parallel here, but applies to in-phase work)
- Stage as separate PRs: schema PR → SDK PR → frontend PR (in series, with deploy between)

The shape of plan-then-execute doesn't change; the constraints around it tighten.

## Anti-patterns

1. **Skipping context bridging.** SDK updater without backend's envelope = guessing.
2. **Parallelizing across layers.** Frontend can't compile until SDK ships. Don't try.
3. **Letting layer N modify layer N-1's files.** SDK updater shouldn't edit the schema. If it discovers an issue, it should `status: "failed"` and bounce back.
4. **One mega sub-agent for all layers.** Defeats the layer separation. Each layer needs its own conventions in its own CLAUDE.md.
5. **Reviewer per layer instead of cross-layer.** Per-layer review can miss inconsistencies (e.g. backend says optional, SDK marks required, frontend never null-checks).

## When this pattern is wrong

- Single-layer task → use orchestrator-worker or just dispatch one implementer
- Layers truly independent → parallel-exploration may fit
- Task too small for layer separation → handle directly in main agent
- Layers don't have stable contracts (early prototyping) → defer this pattern until the contracts settle

## Inverting: when "outside-in" makes sense

Sometimes the frontend's UX dictates the schema, not vice-versa. In that case:
- Phase 1: design the UX contract (a doc, not code)
- Phase 2: backend implementer reads the contract, builds schema to fit
- Phase 3: SDK regenerates
- Phase 4: frontend wires up

Same pattern, different ordering. The DRY onion is "innermost stable thing first" — usually backend, but if the constraint is a designed UX, the design becomes the inner layer.
