# AI Governance Review Playbook

> Operational Playbook — the recurring process that operationalizes [EDR-0005's](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md) risk classification and approval gate.

---

## Purpose

EDR-0005 decides that every AI use case needs a risk classification and an accountable owner before production access. This playbook is the recurring procedure that makes that decision operational — a gate that exists only on paper, with no scheduled process behind it, enforces nothing.

## Roles

| Role | Responsibility |
|---|---|
| Governance Reviewer | Confirms or corrects a use case's self-proposed risk classification; independent of the building team, per EDR-0005's Self-Classification Drift mitigation |
| Approval Authority | Signs off on high-risk use cases before production access is granted; distinct from the Governance Reviewer to avoid one person both classifying and approving their own classification |
| Accountable Owner | The named individual responsible for a specific use case's behavior, incident response, and periodic re-review, per EDR-0005's Decision |
| Governance Program Lead | Maintains the Risk Classification Criteria, tracks approval latency, and owns the offboarding handoff process |

A single person may hold the Governance Reviewer and Governance Program Lead roles in a small organization; the Approval Authority for high-risk use cases should not also be the Governance Reviewer who classified that same use case, since that collapses the independent-review property EDR-0005 requires.

## Recurring Procedures

### 1. Use-Case Intake and Classification

**Trigger:** a team proposes a new AI use case for production deployment.

**Steps:**

1. The building team proposes a risk classification against the criteria in EDR-0005's Risk Classification Criteria table, and names a proposed accountable owner.
2. A Governance Reviewer, independent of the building team, confirms or corrects the classification — per EDR-0005, the highest applicable criterion governs, not an average of the use case's typical behavior.
3. For Low or Moderate classification, the confirmed classification and named owner are recorded, and the use case proceeds to registration (Step 4 below) without further approval.
4. For High classification, the use case is routed to the Approval Authority before registration.
5. Registration: the use case's identity, classification, and owner are recorded in the governance record (per EDR-0005's Observability Requirements) — this record is what the routing layer (EDR-0002) and tool-authorization layer (EDR-0003) check before granting access.

**Output:** a registered use case with a confirmed classification, named owner, and (if High) recorded approval.

### 2. High-Risk Approval

**Trigger:** a use case classified High by intake (Procedure 1) or by reclassification (Procedure 3).

**Steps:**

1. Approval Authority reviews the use case against EDR-0005's Decision Drivers — does the accountable owner exist and is it a specific individual, is the risk classification correctly floored by its highest applicable criterion, does the use case have a defined incident-response path per the [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)?
2. Approve, reject, or request changes. A rejection or change request returns to the building team with the specific gap, not a generic denial.
3. Approval is recorded with date and approver identity, per EDR-0005's Observability Requirements.

**Output:** a recorded approval (or rejection) tied to a specific use case and approver.

### 3. Use-Case Reclassification

**Trigger:** a registered use case changes in a way that could raise its classification — a new tool grant, a new data source, a new user-facing surface — or a fixed periodic review interval is reached, whichever comes first.

**Steps:**

1. The accountable owner (or the change's proposer) flags the change against the Risk Classification Criteria table.
2. A Governance Reviewer confirms whether the change raises the classification.
3. If raised to High and the use case was not previously High, it is routed to High-Risk Approval (Procedure 2) before the change goes live — reclassification does not retroactively grandfather an already-live change.

**Output:** an updated classification with its trigger recorded, per EDR-0005's reclassification-history requirement.

### 4. Ownership Handoff

**Trigger:** an accountable owner is offboarding, changing roles, or otherwise stepping away from a use case.

**Steps:**

1. Offboarding process includes a mandatory check for any use case where the departing person is the accountable owner.
2. A new accountable owner is named and confirmed before the departure takes effect — not on a best-effort basis afterward.
3. If no new owner is named by the departure date, the use case is treated as equivalent to unclassified for enforcement purposes (per EDR-0005's corresponding failure mode), and access is suspended pending a new owner.

**Output:** either a confirmed new owner or a suspended use case — never a use case left with a stale or absent owner.

## Escalation Paths

| Situation | Escalate To | Response Target |
|---|---|---|
| High-risk approval queue growing beyond a defined latency target | Governance Reviewer → Governance Program Lead | Same week — addressed by adding reviewer capacity, per EDR-0005's Approval Bottleneck mitigation, not by loosening criteria |
| A use case is found operating without a registered classification | Governance Program Lead, treated as an incident per the [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md) | Immediate |
| Disagreement between building team and Governance Reviewer on classification | Governance Program Lead adjudicates | Before the use case proceeds — not resolved by the building team escalating around the reviewer |
| Accountable owner departure with no successor named by the departure date | Governance Program Lead, use case suspended | Immediate |

## Anti-Patterns

- Treating classification as a one-time checkbox at launch rather than a live property re-evaluated on material change, per EDR-0005's Decision.
- Letting the Approval Authority and Governance Reviewer be the same person for the same use case, collapsing the independent-review property the gate depends on.
- Allowing a use case to continue running after its accountable owner departs, on the assumption someone will "pick it up eventually" — EDR-0005 treats an unowned use case as equivalent to an unclassified one.

## Related Decisions

- [EDR-0005 — AI Use-Case Risk Classification and Approval Gate](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md)
- [EDR-0002 — Model Routing Strategy](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md)
- [EDR-0003 — Agent Tool-Scope and Permission Boundary](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md)
- [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)
