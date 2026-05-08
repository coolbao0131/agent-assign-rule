# Example — Code Review Handoff

Implementer writes code; reviewer (read-only) audits the diff. Strict separation of concerns enforced by tool allowlists. **This example also demonstrates Q2 in action** — auth-adjacent change requires Opus + mandatory security review + staged PRs.

## Scenario

> "Add input validation to the signup handler. Make sure it's reviewed before merge."

Signup handles authentication-adjacent input. Per the decision tree's Q2 (`references/01-decision-tree.md`), this is high-risk:
- It's auth-adjacent → Q2 fires
- Q2 sets mandatory constraints: **Opus primary, mandatory `code-reviewer` AND `security-auditor` handoff, never parallelize, stage as separate PRs**
- The change might LOOK small, but Q2 doesn't care about size — it cares about category

This example shows the Q2 path. For non-Q2 review handoffs (e.g. cosmetic refactor with reviewer pass), the same shape applies but main agent can be Sonnet and the security-auditor step is optional.

## Setup

`.claude/agents/`:
- `backend-implementer.md` — Sonnet, full tools (the implementer can be Sonnet; only the orchestrator needs Opus under Q2)
- `code-reviewer.md` — Sonnet, **Read/Grep/Glob only**
- `security-auditor.md` — Opus, **Read/Grep/Glob only** (adversarial threat modeling — see `agent-templates/security-auditor.md`)

Main agent: **Opus** (Q2 requires Opus orchestrator for high-risk paths).

## Flow

```
[User] → [Main agent (Opus — required by Q2)]
              │
              ▼
        ┌──── backend-implementer (Sonnet)
        │     - Adds validation
        │     - Updates tests
        │     - Returns: {status, files, tests_passing}
        │
        ▼
[Main validates envelope; status: "complete" + tests_passing: true]
        │
        ├──┐
        │  ├── code-reviewer (Sonnet, read-only) — quality / correctness pass
        │  └── security-auditor (Opus, read-only) — adversarial threat model
        │
        ▼
   (both reviewers run; main reads BOTH reports)
        │
        ▼
[Main shows both reports to user; asks before proceeding]
[Merge as separate PRs per Q2: validation PR first, follow-up if reviewers find issues]
```

The two reviewers run as separate dispatches in the same orchestrator turn. They read the same diff but apply different lenses; their outputs are aggregated by the main agent.

## Why the tool allowlist matters

The code-reviewer template has:
```yaml
tools: Read, Grep, Glob
```

Crucially, no `Edit`, no `Write`, no `Bash`. This is enforced at the tool layer, not by polite request. Even if the reviewer "wants" to fix something it found, it cannot.

That constraint is the entire value of the role. A reviewer that can fix code is just another implementer with a different system prompt — and the user gets no second pair of eyes.

If the reviewer needs to verify a claim by running a test, it CAN'T. It must report "I couldn't verify; suggest running test X." That's the right outcome — the orchestrator (or implementer) runs the test, not the reviewer.

> See `references/06-tool-allowlist.md` for the canonical enforcement rationale.

## Concrete dispatches

### Implementer dispatch
Standard 5-section prompt. Covered in `references/04-prompt-template.md`.

### Reviewer dispatch
```markdown
You are working on: code review of recent signup handler changes
Read first:
- The diff: paths in implementer's envelope below
- src/auth-service/CLAUDE.md
- .claude/rules/code-review.md

The implementer's envelope (verbatim):
{
  "status": "complete",
  "files": ["src/auth-service/handlers/signup.handler.ts",
            "src/auth-service/handlers/__tests__/signup.handler.spec.ts"],
  "tests_passing": true,
  "tests_added": 4,
  "headline": "Added input validation to signup; 4 tests"
}

TASK: Review the signup validation changes
Focus on: auth-relevant correctness, security, edge cases.
This is a read-only review. You have no Edit / Write / Bash.

Acceptance criteria:
1. Every finding has file:line and severity (critical|high|medium|low)
2. No invented findings — if zero issues, say so
3. Output uses exact section headers: Summary, Blocking issues, Non-blocking issues, Notes
4. Escalate auth-bypass / DOS risks with "⚠️ Escalate to human"

RULES:
- You have no Edit/Write/Bash. Don't propose code patches; propose direction in prose.
- Distinguish facts from opinions — opinions go in Notes.
- Stay in the changed scope unless adjacent code directly affects correctness.
- Output: markdown report (orchestrator parses by section headers).
```

### Security-auditor dispatch (Q2-mandatory, parallel with code-reviewer)

```markdown
You are working on: adversarial security review of recent signup handler changes
Read first:
- The diff: same paths as the code-reviewer (from implementer's envelope)
- src/auth-service/CLAUDE.md
- .claude/rules/security.md (if present)

The implementer's envelope (verbatim):
{
  "status": "complete",
  "files": ["src/auth-service/handlers/signup.handler.ts",
            "src/auth-service/handlers/__tests__/signup.handler.spec.ts"],
  "tests_passing": true,
  "tests_added": 4,
  "headline": "Added input validation to signup; 4 tests"
}

TASK: Adversarial security review of the signup validation changes
Apply threat modeling, not a checklist. Think like an attacker.

Threat model context:
- Asset: user accounts and the auth boundary they create
- Attacker: unauthenticated user with an HTTP client
- Goals: account takeover, credential stuffing at scale, account enumeration,
  bypassing input validation to reach downstream sinks

Acceptance criteria:
1. Output the exact sections from agent-templates/security-auditor.md
   (Threat model / Attack surface / Findings by severity / Notes / Out of scope)
2. Each finding has file:line + concrete attack input + fix direction
3. Escalate any of: live secret committed, auth bypass, RCE, data exfiltration
4. State your confidence level — what you verified vs what you didn't

RULES:
- You have no Edit/Write/Bash. Findings only.
- Don't include working exploit code; describe the attack class and input shape.
- Always include "Out of scope / not verified" — bound the audit.
- Severity is the attacker's effort × impact, not your subjective worry level.
- Output: markdown report.
```

The two dispatches run in the same orchestrator turn (parallel). Both return markdown reports the orchestrator reads.

## What the reviewer's report looks like

```markdown
## Summary
Validation logic is mostly correct but has two boundary issues. Tests cover
happy path and obvious failures but miss two edge cases I'd want covered
before merge.

## Blocking issues
1. **[high]** signup.handler.ts:34 — Email regex accepts trailing whitespace
   The pattern `/^[^@]+@[^@]+\.[^@]+$/` doesn't anchor whitespace; an input
   like "alice@example.com\n" passes validation but breaks downstream lookups.
   Fix direction: trim before validate, or use a stricter regex with \S.

## Non-blocking issues
1. **[medium]** signup.handler.ts:51 — Password length checked but not entropy
   8-char minimum allows "12345678". Consider entropy check or breached-list lookup.
   This is a defense-in-depth note; current check matches existing convention.

## Notes
- Test file mirrors existing patterns; consistent with __tests__/ conventions.
- Consider adding a regression test for the trailing-whitespace case once fixed.

## Out of scope / not verified
- Did not check rate limiting on this endpoint (separate concern).
- Did not verify whether downstream code handles the trimmed email correctly.
```

## How the main agent handles both reports

The orchestrator reads BOTH the code-reviewer's and the security-auditor's reports and aggregates:

1. Scan both for `Blocking issues` (code-reviewer's section) AND `Critical` / `High` (security-auditor's findings) AND any `⚠️ Escalate` markers.
2. If EITHER has blocking-equivalent findings:
   - Show both reports to the user.
   - Ask: "Address these before merge?" (yes → re-dispatch implementer with the combined feedback; no → escalate).
3. If only `Non-blocking issues` / `Medium` / `Low` exist:
   - Show both reports.
   - Ask: "Address now or defer?"
4. If both clean:
   - Show summaries from each.
   - Confirm merge readiness for the FIRST staged PR (per Q2, this is the validation PR; follow-ups are separate).

The main agent does NOT auto-fix findings. The user decides. **Under Q2, an escalate flag from the security-auditor blocks merge until human review** — even if the code-reviewer is clean.

## Anti-patterns

1. **Reviewer with Edit / Write.** Defeats the role. If you find this in a template, fix the template.
2. **Skipping the reviewer for "small" auth changes.** Small auth changes are the most dangerous — they look harmless and bypass scrutiny.
3. **Implementer reading the reviewer's report and "fixing" autonomously.** Findings often involve judgment; the user should approve first.
4. **Multiple reviewers in series.** One thorough reviewer beats three shallow passes. If you need security AND general review, use two specialized reviewers (code-reviewer + security-auditor) once each, in parallel.
5. **Reviewer that runs tests.** Reviewer reports the absence of test coverage; doesn't run tests. Test execution is implementer or test-runner's job.

## When this pattern is wrong

- Pure prototyping / scratch code → reviewer overhead isn't worth it
- Reviewer's only finding would be "no tests" → just dispatch test-writer first
- Single-line trivial change to non-sensitive code → main agent's own judgment is enough
- High-risk change → escalate the human in addition to the reviewer; reviewer alone isn't sufficient gating

## Layered review

For Q2 (high-risk) work, use layered review:
1. `code-reviewer` for general correctness/quality
2. `security-auditor` for adversarial threat modeling (separate run, separate persona, both read-only)
3. Human review with both reports in hand

This is more cost than a single reviewer pass, but the cost is small relative to the cost of a missed auth bug.
