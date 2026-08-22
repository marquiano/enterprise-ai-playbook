# Engineering Decision Records (EDR)

An Engineering Decision Record captures a significant, hard-to-reverse architectural decision, the alternatives that were rejected, and the trade-offs accepted in making it.

EDRs exist so that a decision is made once, deliberately, and does not need to be re-argued every time it is questioned.

## When to Write One

Write an EDR when a decision:

- is expensive or slow to reverse;
- affects more than one team or system;
- trades one quality attribute for another (e.g. cost vs. quality, autonomy vs. governance);
- is likely to be challenged later without a recorded rationale.

Do not write an EDR for reversible implementation details, local code style, or choices with no meaningful alternative.

## Numbering

EDRs are numbered sequentially, zero-padded to four digits, and never renumbered or reused:

```
docs/engineering-decisions/EDR-NNNN-SHORT-TITLE.md
```

Example: `EDR-0002-MODEL-ROUTING-STRATEGY.md`.

## Status Lifecycle

| Status | Meaning |
|---|---|
| Proposed | Drafted and internally consistent, but not yet validated against a real implementation or case study |
| Accepted | Validated and in effect; new work should conform to it |
| Rejected | Considered and explicitly not adopted; kept for record |
| Superseded | Replaced by a later EDR, which must be referenced under Related Decisions |

A decision should not move to **Accepted** on the strength of the document alone — see [EDR-0002](EDR-0002-MODEL-ROUTING-STRATEGY.md)'s Review Notes for an example of a decision deliberately held at **Proposed** pending evidence.

## Required Sections

An EDR should contain, in this order:

1. **Status** — current lifecycle state.
2. **Context** — the problem and the forces acting on it.
3. **Decision Drivers** — the constraints the decision must satisfy.
4. **Decision** — the decision itself, stated plainly.
5. **Alternatives Considered** — each rejected option, with its advantages and the specific reason it was rejected.
6. **Consequences** — positive and negative, stated honestly.
7. **Failure Modes** — how the decision can fail in production, and the mitigation for each.
8. **Acceptance Criteria** — observable conditions that confirm the decision is correctly implemented.
9. **Reversal Criteria** — the conditions under which this decision should be revisited.
10. **Related Decisions** — links to other EDRs this one depends on, conflicts with, or supersedes.

Sections that do not apply to a simpler decision may be omitted, but **Alternatives Considered**, **Consequences**, and **Reversal Criteria** should not be — an EDR without a rejected alternative or a way to know it was wrong is not defensible.

## Reference Implementations

- [EDR-0001 — Playbook Scope](EDR-0001-PLAYBOOK-SCOPE.md) — a minimal EDR for a low-ambiguity decision.
- [EDR-0002 — Model Routing Strategy](EDR-0002-MODEL-ROUTING-STRATEGY.md) — a full EDR exercising every required section.
