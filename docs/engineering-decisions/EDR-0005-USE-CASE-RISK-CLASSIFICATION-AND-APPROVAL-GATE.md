# EDR-0005 — AI Use-Case Risk Classification and Approval Gate

## Status

Proposed

## Context

The [Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)'s Foundation Layer lists Governance as a cross-cutting capability alongside Security, Evaluation, and Observability, but the artifacts published so far — [EDR-0002](EDR-0002-MODEL-ROUTING-STRATEGY.md), [EDR-0003](EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md), [EDR-0004](EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md) — all decide *how* a given AI use case behaves once it exists. None of them decide whether a use case is permitted to reach production in the first place, or who is accountable for it once it does.

Without an explicit gate, use cases accumulate through whichever path is easiest: a team with API access builds and ships a workload without anyone outside that team ever assessing its risk, or without any single person accountable for its behavior once it is live. This is not a hypothetical gap — it is the default outcome of not deciding otherwise, since nothing in EDR-0002 through EDR-0004 requires a use case to be classified before those decisions even become relevant to it.

The organization must decide whether a new AI use case requires risk classification and an accountable owner before production deployment, and if so, what that gate consists of.

## Decision Drivers

The governance gate must:

- classify every use case's risk before it reaches production, not as a retrospective audit;
- assign a single accountable owner per use case — accountability distributed across "the team" is accountability held by no one;
- require human approval specifically for high-risk use cases, proportionate to consequence, mirroring the risk-tiered approach already established in [EDR-0003](EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md);
- avoid becoming a universal bottleneck — a low-risk use case should not wait on the same review depth as a high-risk one;
- produce an auditable record of who classified a use case, who approved it, and on what basis;
- apply uniformly regardless of which team or individual builds the use case — an internal prototype from a well-resourced team is not exempt because of who built it.

## Decision

Adopt a mandatory risk-classification and accountability gate for any AI use case before it is granted production model routing (per [EDR-0002](EDR-0002-MODEL-ROUTING-STRATEGY.md)) or tool authorization (per [EDR-0003](EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md)).

Every use case must have, before production deployment:

- a risk classification (low / moderate / high), assessed against defined criteria (see Risk Classification Criteria below), not self-declared by the building team without review;
- a named accountable owner — an individual, not a team — responsible for the use case's behavior, incident response, and periodic re-review;
- for high-risk classification, explicit sign-off from a governance reviewer independent of the building team, before production access is granted.

This gate is procedural, enforced through the recurring process defined in the [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md), and structurally through the routing and tool-authorization layers: EDR-0002's routing layer and EDR-0003's tool-authorization layer both require a registered, classified use case identity before granting model or tool access — an unclassified use case has no path to either.

## Risk Classification Criteria

A use case is classified by the highest applicable criterion, not by its typical or average behavior:

| Signal | Classification Floor |
|---|---|
| Any irreversible-or-external-effect tool authorization (per EDR-0003) | High |
| Processes data classified as sensitive (per EDR-0002's data-sensitivity field) | At least Moderate |
| Customer- or partner-facing output with no human review before delivery | At least Moderate |
| Read-only, internal-only, reversible actions, low-sensitivity data | Low |

A use case that changes in a way that would raise its classification (a new tool grant, a new data source) is re-classified at that point, not only at its original launch — classification is a live property, not a one-time label.

## Alternatives Considered

### Alternative A — No Formal Gate, Retrospective Audit Only

Allow use cases to reach production freely; periodically audit what exists and flag concerns after the fact.

**Advantages**

- zero friction for teams building and shipping use cases;
- no governance infrastructure required before launch.

**Rejected Because**

- a retrospective audit finds problems only after a high-risk use case has already been operating unclassified and unowned, which is precisely the exposure this decision exists to close;
- provides no accountable owner to escalate to when an audit does find a problem.

### Alternative B — Fully Decentralized, Team Self-Classification

Each team classifies its own use cases' risk and decides its own approval process, without central review.

**Advantages**

- fast, no central bottleneck;
- classification stays close to the people who understand the use case best.

**Rejected Because**

- a team assessing its own risk has an incentive (deadline pressure, unfamiliarity with the org-wide risk picture) to under-classify — the same self-restraint problem [EDR-0003](EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) already rejected for tool authorization (its Alternative B) applies here to risk classification;
- produces inconsistent classification criteria across teams, making organization-wide risk visibility impossible.

### Alternative C — Centralized Approval for Every Use Case, Regardless of Risk

Route every use case, regardless of classification, through the same centralized governance review before launch.

**Advantages**

- maximally consistent;
- no risk of under-classification since every case gets full scrutiny.

**Rejected Because**

- creates the same approval-fatigue and bottleneck failure already documented in EDR-0003's rejection of full human approval for every tool call — reviewers facing uniform high volume regardless of actual risk habituate to approving quickly;
- low-risk use cases gain no safety benefit from the same review depth as high-risk ones, making the uniform cost pure overhead for the majority of cases.

## Consequences

**Positive**

- every production use case has one identifiable person accountable for it, closing the "responsible team, accountable nobody" gap;
- review effort concentrates on high-risk use cases rather than being spread uniformly thin;
- routing and tool-authorization layers gain a natural enforcement point — an unclassified use case simply has no registered identity to route or authorize against.

**Negative**

- adds a classification and, for high-risk cases, an approval step before any use case can go live, which teams may experience as friction;
- risk classification criteria require ongoing maintenance as new use-case patterns emerge, similar to the tool risk-tier maintenance already required by EDR-0003;
- a single accountable owner is a single point of organizational failure if that person leaves or is unavailable without a defined handoff process.

## Failure Modes

### Self-Classification Drift

A team classifies its own use case as lower-risk than it actually is, either from optimism or to avoid the higher-risk approval step.

Mitigation:

- classification is reviewed by someone outside the building team before it is finalized, per the [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md)'s intake procedure;
- the Risk Classification Criteria table's "highest applicable criterion" rule removes discretion to average down a use case's classification.

### Accountable Owner Departs Without Handoff

The named accountable owner leaves the organization or role, and the use case continues running with no current accountable person.

Mitigation:

- ownership handoff is a mandatory step in offboarding for any owner of a classified use case, tracked the same way access revocation is tracked;
- a use case with no confirmed current owner is treated as equivalent to an unclassified use case for enforcement purposes — see the Reference Architecture's corresponding failure path.

### Approval Bottleneck at High-Risk Tier

High-risk use cases queue for approval long enough that teams route around the gate by mischaracterizing a use case as moderate-risk to avoid the wait.

Mitigation:

- track approval latency and volume for the high-risk tier specifically; a growing queue is addressed by adding reviewer capacity, not by loosening classification criteria;
- misclassification discovered after the fact is treated as the Self-Classification Drift failure mode above, with the same review-based mitigation.

### Gate Bypassed via Direct Infrastructure Access

A use case obtains model or tool access without going through registration and classification, by reaching Infrastructure directly rather than through the routing and authorization layers — the same class of failure the Reference Architecture already documents as "Any Layer → Infrastructure Layer must not exist."

Mitigation:

- credentials for model providers and tools are held only by the routing layer (EDR-0002) and Enterprise Services (EDR-0003) respectively, never distributed to a use case's own runtime — an unregistered use case has no way to reach either.

## Observability Requirements

Each use case's governance record must include:

- risk classification, its basis, and the reviewer who confirmed it;
- accountable owner and confirmation date of current ownership;
- for high-risk use cases, the approver and approval date;
- reclassification history — every change to risk classification, with the trigger that caused it.

## Acceptance Criteria

The decision is considered successfully implemented when:

- no use case obtains production model routing or tool authorization without a recorded risk classification and confirmed accountable owner;
- every high-risk use case has a recorded approval from a reviewer independent of the building team;
- a use case's accountable-owner field is never blank or stale beyond the offboarding handoff window;
- classification changes are recorded with their trigger, not applied silently.

## Reversal Criteria

This decision should be reconsidered if:

- classification and approval overhead is shown, with real operating data, to meaningfully exceed the risk it prevents for the organization's actual use-case mix;
- the organization's use-case volume and risk profile is small enough that informal, ad hoc oversight is demonstrably sufficient;
- an equally strong enforcement mechanism exists elsewhere that makes a dedicated gate redundant.

## Related Decisions

- [EDR-0002 — Model Routing Strategy](EDR-0002-MODEL-ROUTING-STRATEGY.md) — the routing layer is one of the two enforcement points for this gate.
- [EDR-0003 — Agent Tool-Scope and Permission Boundary](EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) — the tool-authorization layer is the other enforcement point, and the precedent for risk-tiered human approval this decision reuses.
- [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md) — the recurring process that operationalizes this decision.

## Review Notes

This decision must be validated against at least one runnable implementation before being marked Accepted, consistent with the review discipline already applied to EDR-0002 through EDR-0004.
