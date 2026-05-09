# Example — Iterative Codex Review (1-3 rounds)

Adversarial validation of a draft design BEFORE shipping. Distinct from Q6 rescue (used when Claude is STUCK) and `examples/three-way-hybrid.md` (used for cost-optimal mass refactor execution). This pattern is for the moment when Claude has produced a draft and wants an independent reviewer to catch what self-review can't see.

> **⚠️ Decision authority (read this before using the pattern)**
>
> Codex's R1/R2/R3 verdicts are **recommendations, not gates**. Claude/user choose final fixes and may reject any Codex objection. The metric is **decision quality** (did the design hold up under real use), not "rounds completed" or "bugs caught". A pattern that catches 12 bugs but introduces 3 new ones via Codex over-claiming has negative ROI.

## Inputs required before dispatch

Before invoking the pattern, the orchestrator must know:
- **Draft path** — what artifact is being reviewed (single file, set of files, or inline content)
- **Intended audience / context** — Codex doesn't know your team's conventions or constraints; surface them in R1
- **Decision being validated** — "is this design correct to ship" vs "should I pick option A or B" need different prompts
- **Known constraints** — privacy boundaries (can the artifact be sent to an external CLI), token budget, time pressure

If any of these are unclear, push for clarification before R1 — running R1 with unclear scope produces noise, not signal.

## Scenario

User has a draft (skill, design doc, architecture, cross-cutting refactor plan) and wants adversarial review before commit. Single-shot review misses contradictions; iterative refinement converges.

## When to use

Use when:
- Expected cost of a missed design flaw > ~3-5 min + 40-50k tokens of review work, AND
- Decision surface is stable enough that feedback won't be immediately invalidated by ongoing changes.

## When this pattern is wrong (consolidated)

- **Routine implementation** — 1-shot dispatch suffices; iterative review is overkill
- **Time-critical** — 3 rounds = 2-5 min latency
- **Well-trodden Claude territory** — Claude has high confidence; adversarial review surfaces noise more than signal
- **Throwaway / prototype work** — low cost-of-error, not worth the cycles
- **Tasks < 50k tokens** — CLI handoff cost > review value
- **Subjective / taste-based criteria** — adversarial review manufactures objections when criteria are unstable
- **Constrained token / privacy budget** — pattern only justified if review value beats cost AND artifact can safely be shared with external CLI
- **Needs domain judgment, not adversarial polishing** — Codex doesn't know your domain; iterative review compounds the gap

## The 1-3 round protocol (variable, not ritual)

**Cap at 3 rounds.** Stop early when stopping rule satisfied. Going beyond 3 means non-convergence — see Advanced section below.

### ROUND 1 — Critical full review (always)

Send the artifact + ruthless-critique framing. Receive a ranked list mixing actionable / on-the-fence / noise.

R1 prompt template:
```markdown
I have a draft <artifact-type> at <path>. Context:
- <intended audience>
- <decision being validated>
- <known constraints>

Please read it and give a critical review. Be ruthless — if the design has
a fundamental flaw, say so.

Reply under 500 words:
- Verdict: ship-as-is / iterate / structural-rethink
- Strengths (max 3)
- Top 5 concerns, ranked. For each: file:section, concrete problem, suggested fix.
- Skip-worthy issues (1 line each — concerns I might worry about but don't actually matter)

Don't be polite. The cost of fixing now is much lower than after I've shipped.
```

After R1: Claude dismisses noise, queues the actionable for fix. **R1 is a candidate list, not a verdict** — Claude decides which are real.

### ROUND 2 — Push back + drill specifics (usually)

R2 prompt template:
```markdown
Round 2. Your R1 returned <verdict> + N concerns. Here's where I stand:

## Where I concede outright (will fix as you said)
- <items>

## Where I'm pushing back

### Pushback A — on <item>
You said <Codex claim>. I disagree because <reason>.
My counter-proposal: <alternative>.
Defend your position OR concede.

### Drill — on <item>
I'm accepting this; need concrete fix wording. Specifically:
<question 1>
<question 2>

## Format
Reply under 400 words. For each pushback, say `concede` or `defend`
with strongest 1-2 sentence reason. For drill, give specific wording.
```

After R2: Claude locks in concrete fix list. R2's purpose is to **expose where Claude and Codex disagree**; pushing back is essential because Codex can over-claim certainty in R1.

### ROUND 3 — Final draft bless (only if needed)

**Stopping rule**: If R2 produced concrete fix wording for all R1 actionable items AND R2 surfaced no new material objections, **STOP after R2**. Do NOT run R3 ritually.

If you do run R3:

R3 prompt template:
```markdown
Round 3, final convergence. Below is the post-fix version
applying R1+R2 conclusions:

<final draft excerpt or change summary>

Three sign-off questions:
(1) Anything in this final spec you'd remove as not worth its weight?
(2) Anything still missing — I'm at risk of repeating R1 errors at the end?
(3) Last call: structural issue I missed across this discussion?

Format: under 250 words. Verdict line first:
BLESS-WRITE-IT / ONE-MORE-FIX / STOP-RECONSIDER
Then 3 question answers, one sentence each.
```

After R3:
- **BLESS-WRITE-IT** → ship
- **ONE-MORE-FIX** → apply the one fix and ship; do NOT run R4 by default
- **STOP-RECONSIDER** → escalate (see Non-convergence below)

## Failure modes and recovery

- **Codex stalls in R2** (just repeats R1 with different wording) → R2 wasn't drilling specifically enough. Re-frame R2 with concrete pushback items, or accept R1 as final and ship.
- **R3 keeps surfacing new fixes** → design has structural issue, not a tweak issue. See Non-convergence below.
- **Codex over-claims** (asserts "documented behavior" without evidence) → R2 must demand citation. If none, treat as opinion not fact.

## Anti-patterns specific to iterative review

1. **Ritual rounds** — running R3 just because the protocol allows it, when R2 already produced a complete fix list. Stop early.
2. **Adversarial verbosity** — Codex's "be ruthless" framing biases toward more objections than warranted. Treat R1 as a CANDIDATE list; you decide which are real.
3. **Authority laundering** — quoting "Codex said BLESS" as if it's permission to ship. Codex doesn't know your full context. The decision is yours.
4. **Survivorship bias** — counting "bugs caught" without counting "bugs introduced by Codex over-claims". Track the latter too. A pattern that catches 5 bugs and introduces 2 has lower ROI than one that catches 3 and introduces 0.

## Real cost

| Rounds | Wall-clock | Codex tokens |
|---|---|---|
| 1 round (R1 only — for triage) | ~30-90s | ~12-16k |
| 2 rounds (typical) | ~2-3 min | ~25-30k |
| 3 rounds | ~3-5 min | ~40-50k |
| Worst case (R3 catches new bugs, escalation) | ~10 min | ~120k |

## Cross-cite

Distinct from but related to:
- `references/03-cross-cli-delegation.md` — general Codex delegation rules; this pattern is one specific use
- Q6 of `references/01-decision-tree.md` — single-shot rescue when Claude is STUCK; iterative review is when Claude is NOT stuck but wants validation
- `examples/three-way-hybrid.md` — Codex as mass-refactor executor in a pipeline; iterative review uses Codex as critic, not executor

---

## Advanced: Non-convergence handling

> If R3 doesn't bless and you're considering R4, read this section. The 1-3 round protocol covers the happy path; this covers the cases where the protocol itself isn't converging.

### Six scenarios (taxonomy)

| # | Scenario | Symptom |
|---|---|---|
| 1 | **Endless tweaking** | Each round produces 1-2 small new objections; counter never zeros (volume-based) |
| 2 | **Codex contradicts itself** | R3 BLESS then a R4 surfaces objections; or R3 ONE-MORE-FIX whose fix opens a new concern |
| 3 | **Genuinely hard problem** | Artifact has real ambiguity (two valid designs, real trade-off). Rounds expose tension but can't resolve |
| 4 | **Domain mismatch** | Codex repeatedly misses domain context (e.g. company-specific conventions). Each round wastes time |
| 5 | **Codex defending bad take** | Codex commits to a position in R1/R2 and won't concede in R3 even with evidence (positional, not nuanced) |
| 6 | **Unstable spec** | Requirements shift between rounds (user changes target mid-flight, upstream changes). Mimics endless tweaking but root cause is different |

### Auto-detection (numeric triggers)

If ANY of these fire, run R4 (per protocol below) instead of shipping at R3:

```
TRIGGER 1: Net-new objection count
  R3 introduces > 2 objections that are net-new vs R1
  Net-new = different file:line OR different concept class
  Refinement of R1 items doesn't count

TRIGGER 2: R3 re-raises an R2-closed item
  R2's normal challenge of R1 doesn't count (that's the protocol working)
  Only counts: R3 surfacing an issue R2 explicitly marked resolved

TRIGGER 3: ONE-MORE-FIX surfaces another concern (most important signal)
  R3 verdict is ONE-MORE-FIX
  Applying the fix reveals a new concern not in R1/R2
  Indicates the problem space is expanding, not closing
```

### R4 protocol (single round only, hard cap — no R5)

R4 is a focused convergence attempt. **NO further rounds after this** — R4 is binary exit.

R4 prompt template:
```markdown
Round 4. Given R1, R2, R3 history, identify the SINGLE most important
unresolved item. Verdict:

  R4-BLESS    — that one item is now resolved; accept and ship
  R4-ESCALATE — that item cannot be resolved within this protocol;
                surface to user

No third option. No R4-ONE-MORE-FIX.
```

Outcomes:
- **R4-BLESS** — accept and ship
- **R4-ESCALATE** — surface to user with the escalation message format below

There is NO R4-ONE-MORE-FIX. At R4 the protocol has consumed three rounds; adding fourth fix attempts just defers escalation. Binary exit is the contract.

### Escalation message format (what the user sees)

When R4 is ESCALATE (or any earlier round is hopelessly stuck), Claude surfaces a structured message. Both formats: structured first (machine-parseable), prose summary, then Codex verbatim collapsed for audit.

```
⚠️ Non-convergence detected after R[3|4]

[Structured]
status: endless-tweaking | codex-contradicts | hard-problem |
        domain-mismatch | codex-defending-bad-take | unstable-spec
trigger: 1 | 2 | 3 | manual
rounds-completed: 3 | 4

[Prose summary, 3-5 sentences]
What R1 raised: <claude paraphrase>
What R2 resolved: <claude paraphrase>
What R3/R4 outcome was: <claude paraphrase>
Why Claude reads this as non-convergent: <one-sentence diagnosis>
What Codex insists on: <claude paraphrase, 1-2 sentences>

[Codex's last response — collapsed/indented for audit]
> <verbatim Codex response from the most recent round>
> <unedited; user can read to verify Claude's paraphrase>

[Options for user]
(a) Ship with current state, accept residual <X>
(b) Switch reviewer (different model, or human)
(c) Restart with different scope or framing
(d) Abandon — design has structural issue, redesign needed

Your call.
```

**Why both formats**: Claude paraphrase alone is a trust problem (user can't audit). Codex verbatim alone is hostile UX (500-word dump). Both gives faithfulness without punishing the reader — the paraphrase leads, the verbatim is available but not forced.

### Per-scenario advisory (final choice is the user's)

| Scenario | Typical user choice |
|---|---|
| Endless tweaking | (a) ship + accept residual |
| Codex contradicts itself | (a) or (b) |
| Genuinely hard problem | (b) human review, or (a) document the trade-off |
| Domain mismatch | (b) human review; abandon the pattern |
| Codex defending bad take | (a) override Codex, ship |
| Unstable spec | (c) freeze spec, restart |

User has final authority. These are advisory only.

### Caveat: state lives in Claude's session

This protocol assumes Claude is the meta-reviewer holding state across rounds — round counts, R1's issue list, R2's resolutions, trigger history. **If Claude's context is lost mid-protocol (e.g. session restart, `/clear`, context compaction), the trigger history is lost too.**

This isn't a design flaw — it's a property of stateful interactive review. In practice, 3-4 rounds run within a single session, so context loss isn't usually an issue. But if you `/clear` between rounds, restart the protocol from R1; trying to "continue" partway is unsafe.
